# M06 — 중복 Sync와 Process Crash

## `test(android): freeze M06 lost-acknowledgment baseline`

diff --git a/TRACK.md b/TRACK.md
index 3f788f8..0637c66 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -167,6 +167,13 @@ That explicit argument runs only the fixed-ID initial UI launcher; the external
 test intentionally kills it and never calls it again after the death boundary. It is not
 reported as a passing JUnit test. No test-only launcher is present in the app APK.
 
+## M06 boundary — baseline accepted
+
+The frozen lost-acknowledgment scenario reproduces M05's inability to acknowledge its
+already-created Item after process death. The single M06 ledger, exact inputs and evidence
+references are in `verification/M06.md`; initial attempt1, repair0/2. Main owns final Android
+verification. Mutation identity and atomic acknowledgment work is now scoped to M06 only.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
index 74a9b38..8e3edf5 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
@@ -12,9 +12,11 @@ import java.util.concurrent.TimeUnit
 /** Initial fixed-ID launcher only; the external controller performs every real UI action. */
 class M05ScenarioInstrumentation : AndroidJUnitRunner() {
     private var externalController = false
+    private var m06 = false
 
     override fun onCreate(arguments: Bundle?) {
-        externalController = arguments?.getString("m05ExternalController") == "true"
+        m06 = arguments?.getString("m06ExternalController") == "true"
+        externalController = m06 || arguments?.getString("m05ExternalController") == "true"
         super.onCreate(arguments)
     }
 
@@ -30,12 +32,14 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
             ) as MainActivity
             val opened = ItemDatabase.open(targetContext)
             database = opened
-            var localTime = 1_700_000_300_000L
-            val store = ItemStore(opened.items(), nextId = { "device-001" }, now = { localTime.also { localTime += 1_000 } })
-            val sync = ItemSync(store, HttpItemRemote(), now = { 1_700_000_300_000L })
+            val initialTime = if (m06) 1_700_000_400_000L else 1_700_000_300_000L
+            var localTime = initialTime
+            val store = ItemStore(opened.items(), nextId = { if (m06) "crash-001" else "device-001" },
+                now = { localTime.also { localTime += 1_000 } })
+            val sync = ItemSync(store, HttpItemRemote(), now = { initialTime })
             runOnMainSync { activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
             waitForIdleSync()
-            sendStatus(1, Bundle().apply { putString("m05", "ready-for-external-ui-controller") })
+            sendStatus(1, Bundle().apply { putString(if (m06) "m06" else "m05", "ready-for-external-ui-controller") })
             // This is not a JUnit test. The required force-stop intentionally terminates it.
             CountDownLatch(1).await(300, TimeUnit.SECONDS)
             finish(Activity.RESULT_CANCELED, Bundle().apply { putString("error", "External controller did not terminate the app within 300 seconds") })
diff --git a/fixture/server.py b/fixture/server.py
index 888aa93..7eb6ce2 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -1,12 +1,23 @@
 #!/usr/bin/env python3
-"""Local M03/M04 HTTP fixture: in-memory canonical Items, fixed zero delay, test controls."""
+"""Local Item fixture with zero delay and the fixed M06 lost-response contract."""
 
 import argparse
+import hashlib
 from http.server import BaseHTTPRequestHandler, HTTPServer
 import json
+import re
 from urllib.parse import unquote
 
 
+def canonical_request(method, path, payload):
+    return json.dumps({"method": method, "path": path, "payload": payload},
+                      sort_keys=True, separators=(",", ":"), ensure_ascii=False)
+
+
+def payload_hash(method, path, payload):
+    return hashlib.sha256(canonical_request(method, path, payload).encode("utf-8")).hexdigest()
+
+
 class Fixture(HTTPServer):
     def __init__(self, address):
         super().__init__(address, Handler)
@@ -22,6 +33,14 @@ class Fixture(HTTPServer):
         self.next_timestamp = 1700000100000
         self.get_failures = 0
         self.get_requests = 0
+        self.results = {}
+        self.applied = 0
+        self.duplicates = 0
+        self.identity_conflicts = 0
+        self.hash_rejections = 0
+        self.mutation_requests = 0
+        self.dropped_responses = 0
+        self.drop_create_id = None
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -35,17 +54,31 @@ class Fixture(HTTPServer):
         return dict(getFailures=self.get_failures, getRequests=self.get_requests,
                     delayMs=0, nextTimestamp=self.next_timestamp)
 
+    def mutation_state(self):
+        return dict(items=self.sorted_items(), nextTimestamp=self.next_timestamp,
+                    applied=self.applied, duplicates=self.duplicates,
+                    identityConflicts=self.identity_conflicts, hashRejections=self.hash_rejections,
+                    mutationRequests=self.mutation_requests, droppedResponses=self.dropped_responses,
+                    dropCreateId=self.drop_create_id,
+                    results=[dict(clientMutationId=identity, **result)
+                             for identity, result in sorted(self.results.items())])
+
 
 class Handler(BaseHTTPRequestHandler):
-    def respond(self, status, payload):
-        body = json.dumps(payload, separators=(",", ":")).encode()
-        self.send_response(status)
-        self.send_header("Content-Type", "application/json")
-        self.send_header("Content-Length", str(len(body)))
-        self.end_headers()
-        self.wfile.write(body)
+    def respond(self, status, payload, delivery="delivered"):
+        body = json.dumps(payload, separators=(",", ":"), ensure_ascii=False).encode("utf-8")
+        if delivery == "dropped":
+            # Commit/result recording already happened; return without writing HTTP bytes.
+            self.close_connection = True
+        else:
+            self.send_response(status)
+            self.send_header("Content-Type", "application/json")
+            self.send_header("Content-Length", str(len(body)))
+            self.end_headers()
+            self.wfile.write(body)
         print(json.dumps({"method": self.command, "path": self.path, "status": status,
-                          "response": payload}), flush=True)
+                          "response": payload, "delivery": delivery,
+                          **getattr(self, "mutation_evidence", {})}), flush=True)
 
     def log_message(self, *_args):
         pass  # Each response has one structured evidence line instead.
@@ -63,6 +96,54 @@ class Handler(BaseHTTPRequestHandler):
         if "completed" in payload and type(payload["completed"]) is not bool:
             raise ValueError("Expected a Boolean")
 
+    def mutation_payload(self):
+        self.server.mutation_requests += 1
+        wire = self.payload() if int(self.headers.get("Content-Length", "0")) else {}
+        data = {key: value for key, value in wire.items() if key not in ("clientMutationId", "payloadHash")}
+        identity, declared = wire.get("clientMutationId"), wire.get("payloadHash")
+        canonical_payload = None if self.command == "DELETE" and not data else data
+        canonical = canonical_request(self.command, self.path, canonical_payload)
+        actual = hashlib.sha256(canonical.encode("utf-8")).hexdigest()
+        self.mutation_evidence = dict(request=wire, canonical=canonical, actualPayloadHash=actual)
+        self.mutation_identity, self.mutation_canonical = identity, canonical
+        if "clientMutationId" in wire or "payloadHash" in wire:
+            if (not isinstance(identity, str) or not identity or not isinstance(declared, str)
+                    or not re.fullmatch(r"[0-9a-fA-F]{64}", declared)):
+                raise ValueError("mutation_identity_required")
+            if declared.lower() != actual:
+                self.server.hash_rejections += 1
+                raise ValueError("payload_hash_mismatch")
+        # Legacy no-metadata requests retain M05 behavior and never enter deduplication.
+        return data
+
+    def replay_if_known(self):
+        original = self.server.results.get(self.mutation_identity)
+        if original is None:
+            return False
+        if original["canonical"] != self.mutation_canonical:
+            self.server.identity_conflicts += 1
+            self.respond(409, {"error": "identity_conflict"})
+        else:
+            self.server.duplicates += 1
+            self.respond(original["status"], original["response"], delivery="replayed")
+        return True
+
+    def applied_response(self, status, payload):
+        # Snapshot the body; later edits must not change a cached original response.
+        frozen = json.loads(json.dumps(payload))
+        self.server.applied += 1
+        if self.mutation_identity is not None:
+            self.server.results[self.mutation_identity] = dict(
+                canonical=self.mutation_canonical,
+                payloadHash=hashlib.sha256(self.mutation_canonical.encode("utf-8")).hexdigest(),
+                status=status, response=frozen,
+            )
+        drop = self.command == "POST" and payload.get("item", {}).get("id") == self.server.drop_create_id
+        if drop:
+            self.server.drop_create_id = None
+            self.server.dropped_responses += 1
+        self.respond(status, frozen, delivery="dropped" if drop else "delivered")
+
     def do_GET(self):
         if self.path == "/items":
             self.server.get_requests += 1
@@ -76,10 +157,26 @@ class Handler(BaseHTTPRequestHandler):
                                "nextTimestamp": self.server.next_timestamp})
         elif self.path == "/__control":
             self.respond(200, self.server.controls())
+        elif self.path == "/__m06":
+            self.respond(200, self.server.mutation_state())
         else:
             self.respond(404, {"error": "not found"})
 
     def do_POST(self):
+        if self.path == "/__m06/reset":
+            try:
+                data = self.payload()
+                if set(data) != {"dropFirstCreate"} or type(data["dropFirstCreate"]) is not bool:
+                    raise ValueError("Expected dropFirstCreate Boolean")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            self.server.reset()
+            self.server.items = {}
+            self.server.next_timestamp = 1700000400000
+            self.server.drop_create_id = "crash-001" if data["dropFirstCreate"] else None
+            self.respond(200, self.server.mutation_state())
+            return
         if self.path == "/__reset":
             self.server.reset()
             self.respond(200, {"items": self.server.sorted_items(),
@@ -105,18 +202,23 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(404, {"error": "not found"})
             return
         try:
-            data = self.payload()
+            data = self.mutation_payload()
             if set(data) != {"id", "title", "completed"}:
                 raise ValueError("Expected id, title, completed")
-            if not isinstance(data["id"], str) or not data["id"] or data["id"] in self.server.items:
+            if not isinstance(data["id"], str) or not data["id"]:
                 raise ValueError("Expected a new nonempty id")
             self.valid_fields(data)
         except (ValueError, TypeError) as error:
             self.respond(400, {"error": str(error)})
             return
+        if self.replay_if_known():
+            return
+        if data["id"] in self.server.items:
+            self.respond(400, {"error": "Expected a new nonempty id"})
+            return
         item = dict(data, version=1, updatedAt=self.server.tick())
         self.server.items[item["id"]] = item
-        self.respond(201, {"item": item})
+        self.applied_response(201, {"item": item})
 
     def item_id(self):
         if not self.path.startswith("/items/"):
@@ -125,30 +227,40 @@ class Handler(BaseHTTPRequestHandler):
         return item_id if item_id in self.server.items else None
 
     def do_PATCH(self):
-        item_id = self.item_id()
-        if item_id is None:
-            self.respond(404, {"error": "not found"})
-            return
         try:
-            data = self.payload()
+            data = self.mutation_payload()
             if not data or not set(data) <= {"title", "completed"}:
                 raise ValueError("Expected changed title/completed")
             self.valid_fields(data)
         except (ValueError, TypeError) as error:
             self.respond(400, {"error": str(error)})
             return
+        if self.replay_if_known():
+            return
+        item_id = self.item_id()
+        if item_id is None:
+            self.respond(404, {"error": "not found"})
+            return
         item = self.server.items[item_id]
         item.update(data, version=item["version"] + 1, updatedAt=self.server.tick())
-        self.respond(200, {"item": item})
+        self.applied_response(200, {"item": item})
 
     def do_DELETE(self):
+        try:
+            if self.mutation_payload():
+                raise ValueError("Expected empty DELETE payload")
+        except (ValueError, TypeError) as error:
+            self.respond(400, {"error": str(error)})
+            return
+        if self.replay_if_known():
+            return
         item_id = self.item_id()
         if item_id is None:
             self.respond(404, {"error": "not found"})
             return
         del self.server.items[item_id]
         self.server.tick()
-        self.respond(200, {"deletedId": item_id})
+        self.applied_response(200, {"deletedId": item_id})
 
 
 if __name__ == "__main__":
@@ -156,7 +268,7 @@ if __name__ == "__main__":
     parser.add_argument("--port", type=int, default=18080)
     args = parser.parse_args()
     with Fixture(("127.0.0.1", args.port)) as fixture:
-        print(f"M03/M04 fixture listening on http://127.0.0.1:{args.port}; delay=0; getFailures=0", flush=True)
+        print(f"Item fixture listening on http://127.0.0.1:{args.port}; delay=0; getFailures=0", flush=True)
         try:
             fixture.serve_forever()
         except KeyboardInterrupt:
diff --git a/fixture/test_server.py b/fixture/test_server.py
index d546f29..bd5e726 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -1,13 +1,119 @@
 import json
+from contextlib import contextmanager
+from http.client import RemoteDisconnected
 from threading import Thread
 import unittest
 from urllib.request import Request, urlopen
 from urllib.error import HTTPError
 
-from server import Fixture
+from server import Fixture, canonical_request, payload_hash
+
+
+@contextmanager
+def m06_fixture():
+    with Fixture(("127.0.0.1", 0)) as server:
+        thread = Thread(target=server.serve_forever, daemon=True)
+        thread.start()
+        try:
+            def request(method, path, payload=None):
+                data = None if payload is None else json.dumps(payload, ensure_ascii=False).encode("utf-8")
+                try:
+                    response = urlopen(Request(f"http://127.0.0.1:{server.server_port}" + path,
+                                               data=data, method=method,
+                                               headers={"Content-Type": "application/json"}), timeout=5)
+                except HTTPError as error:
+                    response = error
+                with response:
+                    raw = response.read().decode("utf-8")
+                    return response.status, json.loads(raw), raw
+            yield request
+        finally:
+            server.shutdown()
+            thread.join(timeout=5)
 
 
 class FixtureContractTest(unittest.TestCase):
+    def test_m06_canonical_vectors_and_delete_null(self):
+        payload = {"id": "crash-001", "title": "Crash safe", "completed": False}
+        self.assertEqual(
+            '{"method":"POST","path":"/items","payload":{"completed":false,"id":"crash-001","title":"Crash safe"}}',
+            canonical_request("POST", "/items", payload),
+        )
+        self.assertEqual("09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba",
+                         payload_hash("POST", "/items", payload))
+        self.assertEqual("3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73",
+                         payload_hash("POST", "/items", dict(payload, title="Different payload")))
+        self.assertEqual('{"method":"DELETE","path":"/items/crash-001","payload":null}',
+                         canonical_request("DELETE", "/items/crash-001", None))
+        self.assertNotEqual(payload_hash("DELETE", "/items/crash-001", {}),
+                            payload_hash("DELETE", "/items/crash-001", None))
+        self.assertEqual(
+            '{"method":"PATCH","path":"/items/é","payload":{"a":{"b":false,"z":null},"title":"한글 🐈\\n\\\"\\\\","z":[1,true]}}',
+            canonical_request("PATCH", "/items/é", {"z": [1, True], "title": '한글 🐈\n"\\',
+                                                     "a": {"z": None, "b": False}}),
+        )
+
+    def test_m06_lost_ack_replays_original_and_legacy_request_remains_unacknowledged(self):
+        payload = {"id": "crash-001", "title": "Crash safe", "completed": False}
+        wire = dict(payload, clientMutationId="m06-create-001",
+                    payloadHash="09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba")
+        item = dict(payload, version=1, updatedAt=1700000400000)
+        with m06_fixture() as request:
+            request("POST", "/__m06/reset", {"dropFirstCreate": True})
+            with self.assertRaises(RemoteDisconnected):
+                request("POST", "/items", wire)
+            state = request("GET", "/__m06")[1]
+            self.assertEqual((1, 0, 1), (state["applied"], state["duplicates"], state["droppedResponses"]))
+            self.assertEqual([item], state["items"])
+            replay = request("POST", "/items", dict(reversed(list(wire.items()))))
+            self.assertEqual((201, {"item": item}), replay[:2])
+            self.assertEqual(json.dumps({"item": item}, separators=(",", ":")), replay[2])
+            state = request("GET", "/__m06")[1]
+            self.assertEqual((1, 1, 2, 1700000401000),
+                             (state["applied"], state["duplicates"], state["mutationRequests"], state["nextTimestamp"]))
+
+            request("POST", "/__m06/reset", {"dropFirstCreate": True})
+            with self.assertRaises(RemoteDisconnected):
+                request("POST", "/items", payload)
+            self.assertEqual((400, {"error": "Expected a new nonempty id"}),
+                             request("POST", "/items", payload)[:2])
+            state = request("GET", "/__m06")[1]
+            self.assertEqual((1, 0, []), (state["applied"], state["duplicates"], state["results"]))
+            self.assertEqual([item], state["items"])
+
+    def test_m06_identity_collision_hash_validation_and_delete_replay(self):
+        payload = {"id": "crash-001", "title": "Crash safe", "completed": False}
+        wire = dict(payload, clientMutationId="m06-create-001",
+                    payloadHash=payload_hash("POST", "/items", payload))
+        with m06_fixture() as request:
+            request("POST", "/__m06/reset", {"dropFirstCreate": False})
+            original = request("POST", "/items", wire)
+            collision = dict(payload, title="Different payload")
+            self.assertEqual((409, {"error": "identity_conflict"}), request("POST", "/items", dict(
+                collision, clientMutationId="m06-create-001", payloadHash=payload_hash("POST", "/items", collision),
+            ))[:2])
+            self.assertEqual((400, {"error": "payload_hash_mismatch"}), request(
+                "POST", "/items", dict(wire, title="Different payload"),
+            )[:2])
+            self.assertEqual((400, {"error": "mutation_identity_required"}), request(
+                "POST", "/items", dict(payload, clientMutationId="partial"),
+            )[:2])
+            state = request("GET", "/__m06")[1]
+            self.assertEqual((1, 1, 1, 1700000401000),
+                             (state["applied"], state["identityConflicts"], state["hashRejections"], state["nextTimestamp"]))
+            self.assertEqual([original[1]["item"]], state["items"])
+
+            deletion = {"clientMutationId": "m06-delete-001",
+                        "payloadHash": payload_hash("DELETE", "/items/crash-001", None)}
+            deleted = request("DELETE", "/items/crash-001", deletion)
+            self.assertEqual((200, {"deletedId": "crash-001"}), deleted[:2])
+            self.assertEqual(deleted, request("DELETE", "/items/crash-001", deletion))
+            self.assertEqual(original, request("POST", "/items", wire))
+            state = request("GET", "/__m06")[1]
+            self.assertEqual([], state["items"])
+            self.assertEqual((2, 2, 1700000402000),
+                             (state["applied"], state["duplicates"], state["nextTimestamp"]))
+
     def test_frozen_m04_failure_and_remote_change_controls(self):
         with Fixture(("127.0.0.1", 0)) as server:
             thread = Thread(target=server.serve_forever, daemon=True)
diff --git a/verification/M06-inputs.json b/verification/M06-inputs.json
new file mode 100644
index 0000000..db06922
--- /dev/null
+++ b/verification/M06-inputs.json
@@ -0,0 +1,34 @@
+{
+  "thread": "M06",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "9b66fe83f24808ade59d9313a460a3c76652e52c",
+  "seed": [],
+  "mutation": {"sequence": 1, "itemId": "crash-001", "operation": "CREATE", "title": "Crash safe", "completed": false},
+  "clientMutationId": "m06-create-001",
+  "payloadHash": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba",
+  "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Crash safe\"}}",
+  "localItem": {"id": "crash-001", "title": "Crash safe", "completed": false, "version": 0, "updatedAt": 1700000400000},
+  "canonicalItem": {"id": "crash-001", "title": "Crash safe", "completed": false, "version": 1, "updatedAt": 1700000400000},
+  "nextRemoteMutationTimestamp": 1700000400000,
+  "remoteTimestampStep": 1000,
+  "transport": {"fixturePort": 18080, "delayMs": 0, "httpCallTimeoutSeconds": 10, "dropFirstCreateAfterCommit": true},
+  "harness": {"uiWaitSeconds": 30, "adbCommandTimeoutSeconds": 45, "networkWaitSeconds": 30, "preSeedTeardownWaitSeconds": 30, "initialLauncherTimeoutSeconds": 300, "launcherTerminationWaitSeconds": 15},
+  "processBoundary": "Wait for the first Sync to return its visible error and enabled controls, then force-stop; prove old PID absent and plain Activity relaunch uses a different PID with the same durable queue. No install, clear, seed, native write, or instrumentation after this boundary.",
+  "preSeedGate": "Before the one initial launch only, require no app PID and no package ActivityRecord/Task in dumpsys activity activities after force-stop and after pm clear. Bounded observation only; no launch retry.",
+  "baseline": {"schemaVersion": 3, "metadata": "Neither identity field; accepted without deduplication.", "postRestartStatus": 400, "error": "Expected a new nonempty id", "applied": 1, "duplicates": 0, "pending": 1, "acknowledged": null},
+  "fixed": {"schemaVersion": 4, "postRestartStatus": 201, "applied": 1, "duplicates": 1, "pending": 0, "acknowledged": 1},
+  "collision": {"isolated": true, "title": "Different payload", "payloadHash": "3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73", "status": 409, "error": "identity_conflict", "automaticRetries": 0},
+  "wrongDeclaredHash": {"status": 400, "error": "payload_hash_mismatch"},
+  "partialIdentityPair": {"status": 400, "error": "mutation_identity_required"},
+  "deleteVector": {"canonical": "{\"method\":\"DELETE\",\"path\":\"/items/crash-001\",\"payload\":null}", "payloadHash": "443d17b7938e7859a5b26a7d68fc6b5d4a6f9f1006294bdb97439de3f5610535"},
+  "supportingCoverage": [
+    "Canonical JSON is compact literal UTF-8 with recursively sorted keys, standard escaping, integer model values, and JSON null for DELETE; request metadata is excluded.",
+    "Fixture exact replay returns the cached original status/body without another effect; hash validation and an isolated valid-hash collision leave canonical rows unchanged.",
+    "Native Room migration preserves ordered old payloads and assigns durable identity/hash; a new process must reuse stored values.",
+    "Native failures while recording acknowledgment or deleting pending work roll back the entire acknowledgment/dequeue transaction.",
+    "Production synchronization persists terminal identity_conflict and does not automatically resend it; a new session also retains that terminal result.",
+    "All M01-M05 host and actual Android regression methods remain required."
+  ],
+  "scopeLimits": "No baseVersion, scheduler, background execution, push, retry policy, coalescing, or general version-conflict support."
+}
diff --git a/verification/M06.md b/verification/M06.md
new file mode 100644
index 0000000..aee63cf
--- /dev/null
+++ b/verification/M06.md
@@ -0,0 +1,37 @@
+# M06 verification — phase-1
+
+START `9b66fe83f24808ade59d9313a460a3c76652e52c`; branch `track/android-kotlin`.
+SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
+Initial attempt1, repair0/2. Baseline accepted; final candidate not yet verified.
+
+Evidence (`E`): `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M06`.
+Exact arguments/environment/exits are in `E/commands.jsonl` and each named JSON/log.
+Unchanged predecessor files/APK are referenced, not recopied; five changed support files
+and the new test APK are preserved once in `E/baseline-frozen-01`.
+
+| Command | Result |
+| --- | --- |
+| fixture-contract-01, Python unittest | 5/5 pass; original M03/M04 plus canonical hash, lost acknowledgment/replay, collision and DELETE-null contract |
+| baseline-build-01, Gradle assembleDebugAndroidTest | Pass; test-only M06 fixed-ID/clock launcher, unchanged M05 product |
+| baseline-static-freeze-01 | Pass;39 source hashes, actual runner registration, both APKs |
+| baseline-android-01, frozen lost_ack_restart.py limitation/schema3 | Pass expected limitation;71 adb calls, four native DB/WAL captures |
+| baseline-fixture-stop-01 and baseline-audit-01 | Owned PID77827 SIGINT/exit0; PID absent, port18080 free; app absent; network0/1/1 active113 |
+
+Baseline manifest SHA256 `72c90004271829457a198e4d6a406b0aeb75f3ee33af0428afefb7fdeea68552`.
+Inputs SHA256 `8762257a96c0ca6c8b826aa77d74edb0a106e167cbf2081716bbe71f359ac696`.
+App remains verified M05 `0a0cf8888bef40a382171c34bc4d5ce85b2b17b7b1af49edc894743cee31f006`;
+test APK `abe8c2239548a03989d64f3e095b04c0503cbf5993b45f67f47f42329cd85e2b`.
+
+The actual UI creates crash-001/Crash safe. The first explicit Sync commits remotely at
+1700000400000/version1 and loses its HTTP response. Its visible error and enabled controls
+precede PID **16029 → absent → 16331**. Plain Activity relaunch retains the schema3 local
+version0 row and one exact CREATE intent. One explicit replay returns400: the legacy
+request has no identity metadata, so the fixture cannot replay an acknowledgment.
+Remote applied1/duplicates0; local pending1. No install/clear/seed/instrumentation crosses
+the death boundary. The parked initial controller's expected `Process crashed` termination
+is recorded separately; no JUnit suite was claimed. No failure, retry or timeout change.
+
+Root independently accepted the native/PID/UI/HTTP evidence in main
+`threads/evidence/phase-1/android-kotlin/M06/main-baseline-audit.json`.
+`E/baseline-cleanup-audit.json` also checks all39 frozen source files and both APKs.
+Final Android verification belongs to main; no owner-side final suite will run.
diff --git a/verification/lost_ack_restart.py b/verification/lost_ack_restart.py
new file mode 100644
index 0000000..0d388a7
--- /dev/null
+++ b/verification/lost_ack_restart.py
@@ -0,0 +1,253 @@
+#!/usr/bin/env python3
+"""Frozen M06 returned lost response, real process loss, and one explicit replay.
+
+Requires the exclusive emulator lease and an owned frozen fixture on port 18080.
+The initial test launcher supplies fixed constructors only. Every mutation/Sync is
+an actual UI operation; relaunch after death is plain production MainActivity.
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
+
+from offline_queue_restart import OfflineQueueScenario, LAUNCHER
+from process_restart import DB_NAME, PACKAGE, SERIAL
+
+
+INPUTS_PATH = Path(__file__).with_name("M06-inputs.json")
+INPUTS = json.loads(INPUTS_PATH.read_text())
+
+
+class LostAckScenario(OfflineQueueScenario):
+    def __init__(self, args):
+        super().__init__(args)
+        self.result.update(
+            scenario="M06", harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+            inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+            uiHelperSha256=hashlib.sha256(Path(__file__).with_name("offline_queue_restart.py").read_bytes()).hexdigest(),
+            fixtureLog=str(args.fixture_log.resolve()), fixtureLogStartOffset=self.fixture_offset,
+        )
+
+    def pre_seed_teardown(self, label):
+        deadline = time.monotonic() + INPUTS["harness"]["preSeedTeardownWaitSeconds"]
+        while True:
+            pid = self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+            activities = self.adb("shell", "dumpsys", "activity", "activities")
+            remaining = [line.strip() for line in activities.splitlines()
+                         if re.search(r"(?:ActivityRecord\{|Task\{).*" + re.escape(PACKAGE) + r"(?=[/}\s])", line)]
+            if not pid and not remaining:
+                self.result.setdefault("preSeedTeardown", []).append(
+                    {"label": label, "appAbsent": True, "activityAndTaskRecords": [], "lastAdbCommand": self.command_count})
+                return
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"Pre-seed teardown incomplete: pid={pid!r}, records={remaining}")
+            time.sleep(0.2)
+
+    def start_launcher(self):
+        command = [str(self.args.adb), "-s", SERIAL, "shell", "am", "instrument", "-w", "-r",
+                   "-e", "m06ExternalController", "true", LAUNCHER]
+        self.command_count += 1
+        with (self.output / "commands.txt").open("a") as log:
+            log.write(f"{self.command_count:03d} {shlex.join(command)} [parked until external force-stop]\n")
+        self.launcher_files = [(self.output / "launcher.stdout").open("wb"),
+                               (self.output / "launcher.stderr").open("wb")]
+        self.launcher = subprocess.Popen(command, stdout=self.launcher_files[0], stderr=self.launcher_files[1])
+        self.result["initialLauncher"] = {"command": command, "hostPid": self.launcher.pid,
+                                          "purpose": "fixed constructors before seed only; not a JUnit suite"}
+        deadline = time.monotonic() + INPUTS["harness"]["uiWaitSeconds"]
+        while "ready-for-external-ui-controller" not in (self.output / "launcher.stdout").read_text():
+            if self.launcher.poll() is not None or time.monotonic() >= deadline:
+                raise AssertionError("Initial M06 launcher did not become ready")
+            time.sleep(0.2)
+
+    def capture(self, name):
+        rows = self.snapshot(name)
+        folder = self.output / name
+        with tempfile.TemporaryDirectory(prefix="mse-m06-snapshot-") as temporary:
+            local = Path(temporary)
+            for suffix in ("", "-wal"):
+                if (folder / (DB_NAME + suffix)).exists():
+                    shutil.copyfile(folder / (DB_NAME + suffix), local / (DB_NAME + suffix))
+            with sqlite3.connect(local / DB_NAME) as database:
+                columns = [row[1] for row in database.execute("PRAGMA table_info(pending_mutations)")]
+                expected = ["sequence", "itemId", "operation", "title", "completed"]
+                if self.args.expect == "recovered":
+                    expected += ["clientMutationId", "payloadHash", "terminalError"]
+                assert columns == expected, columns
+                pending = [dict(zip(columns, row)) for row in database.execute("SELECT * FROM pending_mutations ORDER BY sequence")]
+                ack_columns = [row[1] for row in database.execute("PRAGMA table_info(acknowledged_mutations)")]
+                acknowledged = None
+                if self.args.expect == "recovered":
+                    assert ack_columns == ["clientMutationId", "payloadHash", "statusCode", "responseBody"], ack_columns
+                    acknowledged = [dict(zip(ack_columns, row)) for row in database.execute(
+                        "SELECT * FROM acknowledged_mutations ORDER BY clientMutationId")]
+                else:
+                    assert ack_columns == [], ack_columns
+                metadata = [list(row) for row in database.execute("SELECT id,lastSuccessfulRefreshAt FROM sync_metadata ORDER BY id")]
+        captured = {"items": rows, "pending": pending, "acknowledged": acknowledged, "refreshMetadata": metadata}
+        (folder / "storage.json").write_text(json.dumps(captured, indent=2) + "\n")
+        (self.output / f"{name}.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        return captured
+
+    def assert_pending_local(self, captured):
+        pending = dict(INPUTS["mutation"])
+        if self.args.expect == "recovered":
+            pending.update(clientMutationId=INPUTS["clientMutationId"], payloadHash=INPUTS["payloadHash"], terminalError=None)
+        assert captured == {
+            "items": [INPUTS["localItem"]], "pending": [pending],
+            "acknowledged": [] if self.args.expect == "recovered" else None, "refreshMetadata": [],
+        }, captured
+
+    def local_ui(self):
+        tree = self.completed_ui([INPUTS["localItem"]["title"]])
+        assert self.matching(tree, **{"content-desc": "Completed: Crash safe", "checked": "false", "enabled": "true"})
+        assert self.matching(tree, text="Pending changes: 1")
+        return tree
+
+    def run(self):
+        self.result["initialNetwork"] = self.wait_network(True)
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        self.pre_seed_teardown("before-install")
+        self.adb("install", "-r", str(self.args.apk.resolve()))
+        self.adb("install", "-r", str(self.args.test_apk.resolve()))
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.pre_seed_teardown("after-clear-before-launch")
+        initial = self.http("/__m06/reset", {"dropFirstCreate": True})
+        assert initial["items"] == [] and initial["applied"] == 0 and initial["duplicates"] == 0
+        assert initial["nextTimestamp"] == INPUTS["nextRemoteMutationTimestamp"]
+        control = self.http("/__control")
+        assert control["getFailures"] == 0 and control["delayMs"] == 0
+        self.start_launcher()
+        self.completed_ui([])
+        self.wait_text("Pending changes: 0")
+        self.tap(**{"class": "android.widget.EditText"})
+        self.text(INPUTS["localItem"]["title"])
+        self.tap(text="Add")
+        self.local_ui()
+        queued = self.capture("before-first-send")
+        self.assert_pending_local(queued)
+        self.result["beforeFirstSend"] = queued
+        assert self.http("/__m06")["items"] == [], "Create was sent without explicit Sync"
+
+        self.tap(text="Sync")
+        self.wait_text("Stale local data · sync error")
+        self.local_ui()  # The first request has returned; controls are enabled before death.
+        before = self.capture("before-stop")
+        self.assert_pending_local(before)
+        remote_before = self.http("/__m06")
+        assert remote_before["items"] == [INPUTS["canonicalItem"]]
+        assert (remote_before["applied"], remote_before["duplicates"], remote_before["mutationRequests"],
+                remote_before["droppedResponses"], remote_before["nextTimestamp"]) == (1, 0, 1, 1, 1700000401000)
+        self.result.update(beforeStop=before, remoteBeforeStop=remote_before, firstFailureReturnedBeforeStop=True)
+
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit(), f"Expected one app PID: {old_pid!r}"
+        assert self.launcher.poll() is None, "Initial launcher ended before the required process boundary"
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True), "Old process remains"
+        self.result["processAbsentAfterForceStop"] = True
+        self.record_launcher_termination(expected=True)
+        # No install, clear, seed, native write, or instrumentation after this boundary.
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and new_pid != old_pid, f"PID not replaced: {old_pid} -> {new_pid}"
+        self.result.update(beforePid=int(old_pid), afterPid=int(new_pid))
+        self.local_ui()
+        self.wait_text("Stale local data")
+        after = self.capture("after-relaunch")
+        assert after == before, f"Durable state changed across process death: {before} -> {after}"
+        self.result["afterRelaunch"] = after
+        assert self.http("/__m06")["mutationRequests"] == 1, "Relaunch automatically resent the mutation"
+
+        self.tap(text="Sync")  # Exactly one explicit foreground replay after restart.
+        if self.args.expect == "recovered":
+            self.wait_text("Fresh local data")
+            self.completed_ui([INPUTS["canonicalItem"]["title"]])
+            self.wait_text("Pending changes: 0")
+        else:
+            self.wait_text("Stale local data · sync error")
+            self.local_ui()
+        final = self.capture("after-replay")
+        remote = self.http("/__m06")
+        assert remote["items"] == [INPUTS["canonicalItem"]]
+        duplicate_count = 1 if self.args.expect == "recovered" else 0
+        assert (remote["applied"], remote["duplicates"], remote["mutationRequests"], remote["droppedResponses"],
+                remote["identityConflicts"], remote["hashRejections"], remote["nextTimestamp"]) == (
+                    1, duplicate_count, 2, 1, 0, 0, 1700000401000)
+        if self.args.expect == "recovered":
+            assert final["items"] == [INPUTS["canonicalItem"]] and final["pending"] == [], final
+            assert len(final["refreshMetadata"]) == 1 and final["refreshMetadata"][0][0] == 1
+            assert final["refreshMetadata"][0][1] > 0
+            original = remote["results"]
+            assert len(original) == 1 and original[0]["clientMutationId"] == INPUTS["clientMutationId"]
+            assert original[0]["canonical"] == INPUTS["canonical"] and original[0]["payloadHash"] == INPUTS["payloadHash"]
+            assert original[0]["status"] == 201 and original[0]["response"] == {"item": INPUTS["canonicalItem"]}
+            assert final["acknowledged"] == [{
+                "clientMutationId": INPUTS["clientMutationId"], "payloadHash": INPUTS["payloadHash"],
+                "statusCode": 201, "responseBody": json.dumps(original[0]["response"], separators=(",", ":"), ensure_ascii=False),
+            }], final
+        else:
+            self.assert_pending_local(final)
+            assert remote["results"] == []
+
+        raw = self.args.fixture_log.read_bytes()[self.fixture_offset:]
+        events = [json.loads(line) for line in raw.splitlines() if line.startswith(b"{")]
+        events = [event for event in events if event["method"] in ("POST", "PATCH", "DELETE")
+                  and event["path"].startswith("/items")]
+        assert len(events) == 2 and all(event["method"] == "POST" and event["path"] == "/items" for event in events)
+        assert (events[0]["status"], events[0]["delivery"]) == (201, "dropped")
+        assert events[0]["canonical"] == events[1]["canonical"] == INPUTS["canonical"]
+        assert events[0]["request"] == events[1]["request"], "Replay changed the persisted request"
+        if self.args.expect == "recovered":
+            assert (events[1]["status"], events[1]["delivery"]) == (201, "replayed")
+            assert events[0]["response"] == events[1]["response"]
+            assert events[1]["request"]["clientMutationId"] == INPUTS["clientMutationId"]
+            assert events[1]["request"]["payloadHash"] == INPUTS["payloadHash"]
+        else:
+            assert (events[1]["status"], events[1]["response"]) == (400, {"error": INPUTS["baseline"]["error"]})
+            assert not {"clientMutationId", "payloadHash"}.intersection(events[0]["request"])
+        self.result.update(afterReplay=final, remoteAfterReplay=remote, mutationHttpStatuses=[event["status"] for event in events],
+                           fixtureLogEndOffset=self.args.fixture_log.stat().st_size,
+                           observed="same durable identity replayed and acknowledged" if self.args.expect == "recovered"
+                           else "M05 server created once, but legacy replay returned 400 and local intent remained pending",
+                           status="PASS")
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("limitation", "recovered"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--test-apk", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--fixture-log", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    scenario = LostAckScenario(parser.parse_args())
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status="FAIL", error=repr(error))
+        raise
+    finally:
+        try:
+            scenario.cleanup()
+        except Exception as error:
+            scenario.result.update(status="FAIL", cleanupError=repr(error))
+            raise
+        finally:
+            scenario.result["adbCommands"] = scenario.command_count
+            (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+            print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()


