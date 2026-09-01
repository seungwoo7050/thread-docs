# M07 — Concurrent Remote Change와 Conflict Resolution

## `test(M07): freeze stale mutation baseline`

diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
index fb01161..1546f03 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
@@ -14,10 +14,12 @@ import java.util.concurrent.TimeUnit
 class M05ScenarioInstrumentation : AndroidJUnitRunner() {
     private var externalController = false
     private var m06 = false
+    private var m07Case: String? = null
 
     override fun onCreate(arguments: Bundle?) {
         m06 = arguments?.getString("m06ExternalController") == "true"
-        externalController = m06 || arguments?.getString("m05ExternalController") == "true"
+        m07Case = arguments?.getString("m07Case")?.also { require(it in setOf("A", "B", "C")) }
+        externalController = m07Case != null || m06 || arguments?.getString("m05ExternalController") == "true"
         super.onCreate(arguments)
     }
 
@@ -33,15 +35,28 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
             ) as MainActivity
             val opened = ItemDatabase.open(targetContext)
             database = opened
-            val initialTime = if (m06) 1_700_000_400_000L else 1_700_000_300_000L
+            val initialTime = when {
+                m07Case != null -> 1_700_000_500_000L
+                m06 -> 1_700_000_400_000L
+                else -> 1_700_000_300_000L
+            }
             var localTime = initialTime
             val store = ItemStore(opened.items(), nextId = { if (m06) "crash-001" else "device-001" },
                 now = { localTime.also { localTime += 1_000 } },
-                nextMutationId = { if (m06) "m06-create-001" else UUID.randomUUID().toString() })
+                nextMutationId = {
+                    when (m07Case) {
+                        "A" -> "m07-update-001"
+                        "B" -> "m07-delete-001"
+                        "C" -> "m07-deleted-001"
+                        else -> if (m06) "m06-create-001" else UUID.randomUUID().toString()
+                    }
+                })
             val sync = ItemSync(store, HttpItemRemote(), now = { initialTime })
             runOnMainSync { activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
             waitForIdleSync()
-            sendStatus(1, Bundle().apply { putString(if (m06) "m06" else "m05", "ready-for-external-ui-controller") })
+            sendStatus(1, Bundle().apply {
+                putString(if (m07Case != null) "m07" else if (m06) "m06" else "m05", "ready-for-external-ui-controller")
+            })
             // This is not a JUnit test. The required force-stop intentionally terminates it.
             CountDownLatch(1).await(300, TimeUnit.SECONDS)
             finish(Activity.RESULT_CANCELED, Bundle().apply { putString("error", "External controller did not terminate the app within 300 seconds") })
diff --git a/fixture/server.py b/fixture/server.py
index 7eb6ce2..99f4262 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -1,5 +1,5 @@
 #!/usr/bin/env python3
-"""Local Item fixture with zero delay and the fixed M06 lost-response contract."""
+"""Local Item fixture with the fixed M06 replay and M07 version-conflict contracts."""
 
 import argparse
 import hashlib
@@ -41,6 +41,11 @@ class Fixture(HTTPServer):
         self.mutation_requests = 0
         self.dropped_responses = 0
         self.drop_create_id = None
+        self.tombstones = {}
+        self.m07_enabled = False
+        self.version_conflicts = 0
+        self.remote_mutations = 0
+        self.requests = []
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -50,6 +55,15 @@ class Fixture(HTTPServer):
     def sorted_items(self):
         return [self.items[key] for key in sorted(self.items)]
 
+    def sorted_tombstones(self):
+        return [self.tombstones[key] for key in sorted(self.tombstones)]
+
+    def read_state(self):
+        state = dict(items=self.sorted_items())
+        if self.m07_enabled:
+            state["tombstones"] = self.sorted_tombstones()
+        return state
+
     def controls(self):
         return dict(getFailures=self.get_failures, getRequests=self.get_requests,
                     delayMs=0, nextTimestamp=self.next_timestamp)
@@ -63,6 +77,17 @@ class Fixture(HTTPServer):
                     results=[dict(clientMutationId=identity, **result)
                              for identity, result in sorted(self.results.items())])
 
+    def conflict_state(self):
+        return dict(self.mutation_state(), tombstones=self.sorted_tombstones(),
+                    versionConflicts=self.version_conflicts, remoteMutations=self.remote_mutations,
+                    requests=self.requests,
+                    conflicts=[request for request in self.requests if request["status"] == 409])
+
+    def delete_item(self, item_id):
+        item = self.items.pop(item_id)
+        self.tombstones[item_id] = dict(id=item_id, version=item["version"] + 1,
+                                       updatedAt=self.tick(), deleted=True)
+
 
 class Handler(BaseHTTPRequestHandler):
     def respond(self, status, payload, delivery="delivered"):
@@ -76,9 +101,12 @@ class Handler(BaseHTTPRequestHandler):
             self.send_header("Content-Length", str(len(body)))
             self.end_headers()
             self.wfile.write(body)
-        print(json.dumps({"method": self.command, "path": self.path, "status": status,
-                          "response": payload, "delivery": delivery,
-                          **getattr(self, "mutation_evidence", {})}), flush=True)
+        evidence = {"method": self.command, "path": self.path, "status": status,
+                    "response": payload, "delivery": delivery,
+                    **getattr(self, "mutation_evidence", {})}
+        if hasattr(self, "mutation_evidence"):
+            self.server.requests.append(json.loads(json.dumps(evidence)))
+        print(json.dumps(evidence), flush=True)
 
     def log_message(self, *_args):
         pass  # Each response has one structured evidence line instead.
@@ -113,6 +141,8 @@ class Handler(BaseHTTPRequestHandler):
             if declared.lower() != actual:
                 self.server.hash_rejections += 1
                 raise ValueError("payload_hash_mismatch")
+        if "baseVersion" in data and type(data["baseVersion"]) is not int:
+            raise ValueError("Expected integer baseVersion")
         # Legacy no-metadata requests retain M05 behavior and never enter deduplication.
         return data
 
@@ -128,22 +158,40 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(original["status"], original["response"], delivery="replayed")
         return True
 
-    def applied_response(self, status, payload):
+    def cache_response(self, status, payload):
         # Snapshot the body; later edits must not change a cached original response.
         frozen = json.loads(json.dumps(payload))
-        self.server.applied += 1
         if self.mutation_identity is not None:
             self.server.results[self.mutation_identity] = dict(
                 canonical=self.mutation_canonical,
                 payloadHash=hashlib.sha256(self.mutation_canonical.encode("utf-8")).hexdigest(),
                 status=status, response=frozen,
             )
+        return frozen
+
+    def applied_response(self, status, payload):
+        frozen = self.cache_response(status, payload)
+        self.server.applied += 1
         drop = self.command == "POST" and payload.get("item", {}).get("id") == self.server.drop_create_id
         if drop:
             self.server.drop_create_id = None
             self.server.dropped_responses += 1
         self.respond(status, frozen, delivery="dropped" if drop else "delivered")
 
+    def version_conflict_if_stale(self, data):
+        if "baseVersion" not in data or not self.path.startswith("/items/"):
+            return False
+        item_id = unquote(self.path[len("/items/"):])
+        item = self.server.items.get(item_id)
+        tombstone = self.server.tombstones.get(item_id)
+        current = item or tombstone
+        if current is None or current["version"] == data["baseVersion"]:
+            return False
+        self.server.version_conflicts += 1
+        response = self.cache_response(409, dict(error="version_conflict", item=item, tombstone=tombstone))
+        self.respond(409, response)
+        return True
+
     def do_GET(self):
         if self.path == "/items":
             self.server.get_requests += 1
@@ -151,18 +199,54 @@ class Handler(BaseHTTPRequestHandler):
                 self.server.get_failures -= 1
                 self.respond(503, {"error": "temporary failure"})
             else:
-                self.respond(200, {"items": self.server.sorted_items()})
+                self.respond(200, self.server.read_state())
         elif self.path == "/__state":
-            self.respond(200, {"items": self.server.sorted_items(),
-                               "nextTimestamp": self.server.next_timestamp})
+            self.respond(200, dict(self.server.read_state(), nextTimestamp=self.server.next_timestamp))
         elif self.path == "/__control":
             self.respond(200, self.server.controls())
         elif self.path == "/__m06":
             self.respond(200, self.server.mutation_state())
+        elif self.path == "/__m07":
+            self.respond(200, self.server.conflict_state())
         else:
             self.respond(404, {"error": "not found"})
 
     def do_POST(self):
+        if self.path == "/__m07/reset":
+            try:
+                if self.payload():
+                    raise ValueError("Expected empty reset object")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            self.server.reset()
+            self.server.m07_enabled = True
+            self.server.items = {
+                "conflict-001": dict(id="conflict-001", title="Initial", completed=False,
+                                     version=1, updatedAt=1700000000000),
+            }
+            self.server.next_timestamp = 1700000500000
+            self.respond(200, self.server.conflict_state())
+            return
+        if self.path == "/__m07/remote":
+            try:
+                data = self.payload()
+                if set(data) != {"operation"} or data["operation"] not in ("rename", "delete"):
+                    raise ValueError("Expected operation rename or delete")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            item = self.server.items.get("conflict-001")
+            if not self.server.m07_enabled or self.server.remote_mutations or item is None or item["version"] != 1:
+                self.respond(409, {"error": "remote_control_requires_initial_item"})
+                return
+            if data["operation"] == "rename":
+                item.update(title="Remote winner", version=2, updatedAt=self.server.tick())
+            else:
+                self.server.delete_item("conflict-001")
+            self.server.remote_mutations += 1
+            self.respond(200, self.server.conflict_state())
+            return
         if self.path == "/__m06/reset":
             try:
                 data = self.payload()
@@ -229,37 +313,42 @@ class Handler(BaseHTTPRequestHandler):
     def do_PATCH(self):
         try:
             data = self.mutation_payload()
-            if not data or not set(data) <= {"title", "completed"}:
+            changes = {key: value for key, value in data.items() if key != "baseVersion"}
+            if not changes or not set(changes) <= {"title", "completed"}:
                 raise ValueError("Expected changed title/completed")
-            self.valid_fields(data)
+            self.valid_fields(changes)
         except (ValueError, TypeError) as error:
             self.respond(400, {"error": str(error)})
             return
         if self.replay_if_known():
             return
+        if self.version_conflict_if_stale(data):
+            return
         item_id = self.item_id()
         if item_id is None:
             self.respond(404, {"error": "not found"})
             return
         item = self.server.items[item_id]
-        item.update(data, version=item["version"] + 1, updatedAt=self.server.tick())
+        item.update(changes, version=item["version"] + 1, updatedAt=self.server.tick())
         self.applied_response(200, {"item": item})
 
     def do_DELETE(self):
         try:
-            if self.mutation_payload():
+            data = self.mutation_payload()
+            if not set(data) <= {"baseVersion"}:
                 raise ValueError("Expected empty DELETE payload")
         except (ValueError, TypeError) as error:
             self.respond(400, {"error": str(error)})
             return
         if self.replay_if_known():
             return
+        if self.version_conflict_if_stale(data):
+            return
         item_id = self.item_id()
         if item_id is None:
             self.respond(404, {"error": "not found"})
             return
-        del self.server.items[item_id]
-        self.server.tick()
+        self.server.delete_item(item_id)
         self.applied_response(200, {"deletedId": item_id})
 
 
diff --git a/fixture/test_server.py b/fixture/test_server.py
index bd5e726..eb4c3b3 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -32,7 +32,203 @@ def m06_fixture():
             thread.join(timeout=5)
 
 
+def mutation_wire(method, path, payload, identity):
+    return dict(payload or {}, clientMutationId=identity,
+                payloadHash=payload_hash(method, path, payload))
+
+
 class FixtureContractTest(unittest.TestCase):
+    def test_m07_fixed_conflicts_and_fresh_explicit_edit(self):
+        path = "/items/conflict-001"
+        seed = dict(id="conflict-001", title="Initial", completed=False,
+                    version=1, updatedAt=1700000000000)
+        winner = dict(seed, title="Remote winner", version=2, updatedAt=1700000500000)
+        tombstone = dict(id="conflict-001", version=2, updatedAt=1700000500000, deleted=True)
+        with m06_fixture() as request:
+            for case, method, identity, remote in (
+                ("A", "PATCH", "m07-update-001", "rename"),
+                ("B", "DELETE", "m07-delete-001", "rename"),
+                ("C", "PATCH", "m07-deleted-001", "delete"),
+            ):
+                with self.subTest(case=case):
+                    reset = request("POST", "/__m07/reset", {})
+                    self.assertEqual(200, reset[0])
+                    self.assertEqual(([seed], [], [], [], [], 0, 0, 0, 1700000500000),
+                                     (reset[1]["items"], reset[1]["tombstones"], reset[1]["requests"],
+                                      reset[1]["conflicts"], reset[1]["results"], reset[1]["applied"],
+                                      reset[1]["versionConflicts"], reset[1]["remoteMutations"],
+                                      reset[1]["nextTimestamp"]))
+                    self.assertEqual((200, {"items": [seed], "tombstones": []}),
+                                     request("GET", "/items")[:2])
+                    remote_state = request("POST", "/__m07/remote", {"operation": remote})
+                    self.assertEqual(200, remote_state[0])
+                    self.assertEqual((1, 0, 0, [], 1700000501000),
+                                     (remote_state[1]["remoteMutations"], remote_state[1]["applied"],
+                                      remote_state[1]["mutationRequests"], remote_state[1]["results"],
+                                      remote_state[1]["nextTimestamp"]))
+                    payload = {"baseVersion": 1}
+                    if method == "PATCH":
+                        payload["title"] = "Local attempt"
+                    wire = mutation_wire(method, path, payload, identity)
+                    conflict = dict(error="version_conflict", item=None if case == "C" else winner,
+                                    tombstone=tombstone if case == "C" else None)
+                    self.assertEqual((409, conflict), request(method, path, wire)[:2])
+                    state = request("GET", "/__m07")[1]
+                    canonical = {"items": [] if case == "C" else [winner],
+                                 "tombstones": [tombstone] if case == "C" else []}
+                    self.assertEqual((200, canonical), request("GET", "/items")[:2])
+                    self.assertEqual((0, 1, 1, 1, 0, 1700000501000),
+                                     (state["applied"], state["remoteMutations"], state["mutationRequests"],
+                                      state["versionConflicts"], state["duplicates"], state["nextTimestamp"]))
+                    self.assertEqual(1, len(state["requests"]))
+                    self.assertEqual(wire, state["requests"][0]["request"])
+                    self.assertEqual(payload, json.loads(state["requests"][0]["canonical"])["payload"])
+                    self.assertEqual(wire["payloadHash"], state["requests"][0]["actualPayloadHash"])
+                    self.assertEqual(state["requests"], state["conflicts"])
+                    self.assertEqual((identity, 409, conflict),
+                                     (state["results"][0]["clientMutationId"], state["results"][0]["status"],
+                                      state["results"][0]["response"]))
+                    if case == "A":
+                        explicit = {"title": "Explicit edit", "baseVersion": 2}
+                        edit = dict(winner, title="Explicit edit", version=3, updatedAt=1700000501000)
+                        self.assertEqual((200, {"item": edit}), request("PATCH", path, mutation_wire(
+                            "PATCH", path, explicit, "m07-explicit-001",
+                        ))[:2])
+                        state = request("GET", "/__m07")[1]
+                        self.assertEqual(([edit], 1, 1, 2, 1, 1700000502000),
+                                         (state["items"], state["applied"], state["remoteMutations"],
+                                          state["mutationRequests"], state["versionConflicts"],
+                                          state["nextTimestamp"]))
+                        results = {result["clientMutationId"]: result for result in state["results"]}
+                        self.assertEqual(conflict, results[identity]["response"])
+                        self.assertEqual({"item": edit}, results["m07-explicit-001"]["response"])
+                        self.assertEqual(conflict, state["conflicts"][0]["response"])
+
+    def test_m07_accepted_patch_replay_precedes_version_check_and_base_only_collision(self):
+        path = "/items/conflict-001"
+        payload = {"title": "Accepted own edit", "baseVersion": 1}
+        wire = mutation_wire("PATCH", path, payload, "m07-accepted-001")
+        with m06_fixture() as request:
+            request("POST", "/__m07/reset", {})
+            original = request("PATCH", path, wire)
+            accepted = dict(id="conflict-001", title="Accepted own edit", completed=False,
+                            version=2, updatedAt=1700000500000)
+            self.assertEqual((200, {"item": accepted}), original[:2])
+            successor = {"completed": True, "baseVersion": 2}
+            current = dict(accepted, completed=True, version=3, updatedAt=1700000501000)
+            self.assertEqual((200, {"item": current}), request("PATCH", path, mutation_wire(
+                "PATCH", path, successor, "m07-accepted-002",
+            ))[:2])
+            self.assertEqual(original, request("PATCH", path, dict(reversed(list(wire.items())))))
+            self.assertEqual((400, {"error": "payload_hash_mismatch"}), request(
+                "PATCH", path, dict(wire, baseVersion=2),
+            )[:2])
+            changed_base = dict(payload, baseVersion=2)
+            self.assertNotEqual(wire["payloadHash"], payload_hash("PATCH", path, changed_base))
+            self.assertEqual((409, {"error": "identity_conflict"}), request("PATCH", path, mutation_wire(
+                "PATCH", path, changed_base, "m07-accepted-001",
+            ))[:2])
+            state = request("GET", "/__m07")[1]
+            self.assertEqual(([current], 2, 1, 1, 1, 0, 5, 1700000502000),
+                             (state["items"], state["applied"], state["duplicates"],
+                              state["identityConflicts"], state["hashRejections"], state["versionConflicts"],
+                              state["mutationRequests"], state["nextTimestamp"]))
+            self.assertEqual(["delivered", "delivered", "replayed", "delivered", "delivered"],
+                             [entry["delivery"] for entry in state["requests"]])
+            for invalid_base in (True, "1", 1.0, None):
+                with self.subTest(invalid_base=invalid_base):
+                    self.assertEqual((400, {"error": "Expected integer baseVersion"}), request(
+                        "PATCH", path, mutation_wire("PATCH", path, dict(payload, baseVersion=invalid_base),
+                                                     "m07-invalid-base"),
+                    )[:2])
+            self.assertEqual([current], request("GET", "/__m07")[1]["items"])
+
+    def test_m07_conflicted_identity_is_bound_and_replays_its_original_conflict(self):
+        path = "/items/conflict-001"
+        payload = {"title": "Local attempt", "baseVersion": 1}
+        wire = mutation_wire("PATCH", path, payload, "m07-update-001")
+        with m06_fixture() as request:
+            request("POST", "/__m07/reset", {})
+            request("POST", "/__m07/remote", {"operation": "rename"})
+            original = request("PATCH", path, wire)
+            self.assertEqual(409, original[0])
+            edit = request("PATCH", path, mutation_wire(
+                "PATCH", path, {"title": "Explicit edit", "baseVersion": 2}, "m07-explicit-001",
+            ))
+            self.assertEqual(200, edit[0])
+            self.assertEqual(original, request("PATCH", path, wire))
+            self.assertEqual((409, {"error": "identity_conflict"}), request("PATCH", path, mutation_wire(
+                "PATCH", path, dict(payload, baseVersion=2), "m07-update-001",
+            ))[:2])
+            state = request("GET", "/__m07")[1]
+            self.assertEqual(([edit[1]["item"]], 1, 1, 1, 1, 4, 1700000502000),
+                             (state["items"], state["applied"], state["duplicates"], state["versionConflicts"],
+                              state["identityConflicts"], state["mutationRequests"], state["nextTimestamp"]))
+            self.assertEqual(original[1], state["conflicts"][1]["response"])
+            self.assertEqual("replayed", state["conflicts"][1]["delivery"])
+
+    def test_m07_preserves_m05_four_accepts_and_own_successor_bases(self):
+        gamma = dict(id="device-001", title="Gamma", completed=False, version=1, updatedAt=1700000300000)
+        renamed = dict(id="remote-001", title="Queued edit", completed=False,
+                       version=2, updatedAt=1700000301000)
+        completed = dict(renamed, completed=True, version=3, updatedAt=1700000302000)
+        operations = (
+            ("POST", "/items", {"id": "device-001", "title": "Gamma", "completed": False},
+             201, {"item": gamma}),
+            ("PATCH", "/items/remote-001", {"title": "Queued edit", "baseVersion": 1},
+             200, {"item": renamed}),
+            ("PATCH", "/items/remote-001", {"completed": True, "baseVersion": 2},
+             200, {"item": completed}),
+            ("DELETE", "/items/remote-002", {"baseVersion": 1}, 200, {"deletedId": "remote-002"}),
+        )
+        with m06_fixture() as request:
+            request("POST", "/__control", {"nextTimestamp": 1700000300000})
+            for index, (method, path, payload, status, body) in enumerate(operations, 1):
+                self.assertEqual((status, body), request(method, path, mutation_wire(
+                    method, path, payload, f"m05-{index}",
+                ))[:2])
+            self.assertEqual((200, {"items": [gamma, completed]}), request("GET", "/items")[:2])
+            state = request("GET", "/__m07")[1]
+            self.assertEqual((4, 4, 0, 0, 0, 1700000304000),
+                             (state["applied"], state["mutationRequests"], state["duplicates"],
+                              state["versionConflicts"], state["remoteMutations"], state["nextTimestamp"]))
+            self.assertEqual([None, 1, 2, 1],
+                             [entry["request"].get("baseVersion") for entry in state["requests"]])
+            self.assertEqual([201, 200, 200, 200], [entry["status"] for entry in state["requests"]])
+            self.assertEqual([dict(id="remote-002", version=2, updatedAt=1700000303000, deleted=True)],
+                             state["tombstones"])
+            self.assertEqual(4, len(state["results"]))
+
+    def test_m07_no_base_preserves_m06_stale_update_and_delete_limitation(self):
+        path = "/items/conflict-001"
+        with m06_fixture() as request:
+            for case, method, payload, remote, expected_status in (
+                ("A", "PATCH", {"title": "Local attempt"}, "rename", 200),
+                ("B", "DELETE", None, "rename", 200),
+                ("C", "PATCH", {"title": "Local attempt"}, "delete", 404),
+            ):
+                with self.subTest(case=case):
+                    request("POST", "/__m07/reset", {})
+                    request("POST", "/__m07/remote", {"operation": remote})
+                    result = request(method, path, mutation_wire(method, path, payload, "m06-baseline-" + case))
+                    self.assertEqual(expected_status, result[0])
+                    state = request("GET", "/__m07")[1]
+                    self.assertEqual((1 if case != "C" else 0, 1, 1, 0, []),
+                                     (state["applied"], state["remoteMutations"], state["mutationRequests"],
+                                      state["versionConflicts"], state["conflicts"]))
+                    if case == "A":
+                        item = dict(id="conflict-001", title="Local attempt", completed=False,
+                                    version=3, updatedAt=1700000501000)
+                        self.assertEqual({"item": item}, result[1])
+                        self.assertEqual([item], state["items"])
+                    else:
+                        self.assertEqual({"deletedId": "conflict-001"} if case == "B" else {"error": "not found"},
+                                         result[1])
+                        self.assertEqual([], state["items"])
+                        self.assertEqual([dict(id="conflict-001", version=3 if case == "B" else 2,
+                                               updatedAt=1700000501000 if case == "B" else 1700000500000,
+                                               deleted=True)], state["tombstones"])
+
     def test_m06_canonical_vectors_and_delete_null(self):
         payload = {"id": "crash-001", "title": "Crash safe", "completed": False}
         self.assertEqual(
diff --git a/verification/M07-inputs.json b/verification/M07-inputs.json
new file mode 100644
index 0000000..1a9d43f
--- /dev/null
+++ b/verification/M07-inputs.json
@@ -0,0 +1,25 @@
+{
+  "thread": "M07",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "425a0fbc05b7802c8e788f52aba59a4274d7a725",
+  "seed": {"id":"conflict-001","title":"Initial","completed":false,"version":1,"updatedAt":1700000000000},
+  "localTitle": "Local attempt",
+  "localTimestamp": 1700000500000,
+  "canonicalItem": {"id":"conflict-001","title":"Remote winner","completed":false,"version":2,"updatedAt":1700000500000},
+  "canonicalTombstone": {"id":"conflict-001","version":2,"updatedAt":1700000500000,"deleted":true},
+  "explicitItem": {"id":"conflict-001","title":"Explicit edit","completed":false,"version":3,"updatedAt":1700000501000},
+  "cases": [
+    {"name":"A","operation":"RENAME","method":"PATCH","clientMutationId":"m07-update-001","remoteOperation":"rename","baselineStatus":200},
+    {"name":"B","operation":"DELETE","method":"DELETE","clientMutationId":"m07-delete-001","remoteOperation":"rename","baselineStatus":200},
+    {"name":"C","operation":"RENAME","method":"PATCH","clientMutationId":"m07-deleted-001","remoteOperation":"delete","baselineStatus":404}
+  ],
+  "conflictUi": "CONFLICT: remote value kept; local attempt retained.",
+  "transport": {"fixturePort":18080,"delayMs":0,"httpCallTimeoutSeconds":10},
+  "harness": {"uiWaitSeconds":30,"adbCommandTimeoutSeconds":45,"networkWaitSeconds":30,"preSeedTeardownWaitSeconds":30,"initialLauncherTimeoutSeconds":300,"launcherTerminationWaitSeconds":15},
+  "processBoundary": "Each independent case completes its actual offline local edit and deterministic remote change, then allows exactly one Sync response. Force-stop after controls are ready, prove the old PID absent, and relaunch plain MainActivity with a different PID and unchanged native durable state. No seed, instrumentation, clear, install, or native write across that boundary.",
+  "preSeedGate": "Before initial test setup only, require PID absence and no own attached Task/Hist ActivityRecord in dumpsys activities. Ignore historical field references, retain exiting attached records as busy, reject malformed output, and retain the 30-second bound. No launch retries.",
+  "baseline": "Unchanged M06 omits baseVersion: A overwrites the newer title at v3; B deletes the newer Item at v3; C receives404 and keeps its optimistic local edit pending. Each result is reopened after actual process loss. No baseline action is retried.",
+  "fixed": "Each original base1 identity is persisted as version_conflict with its original envelope and the correct canonical Item/tombstone. Reopen, then one ordinary drain sends none of those intents. A alone then receives a fresh explicit user edit with a new identity/base2 and canonical v3/time1700000501000.",
+  "scopeLimits": "No automatic retry, coalescing, external rebase, lifecycle restoration, scheduler, push, or merge editor."
+}
diff --git a/verification/M07.md b/verification/M07.md
new file mode 100644
index 0000000..64808e8
--- /dev/null
+++ b/verification/M07.md
@@ -0,0 +1,32 @@
+# M07 — phase-1 version conflicts
+
+Status: baseline independently accepted; implementation/final verification pending.
+START `425a0fbc05b7802c8e788f52aba59a4274d7a725` (verified phase-1 M06).
+SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`; attempt 1, repair 0/2.
+
+## Frozen unchanged-M06 baseline
+
+Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M07`.
+[Manifest](../../evidence/phase-1/android-kotlin/M07/baseline-frozen-01/manifest.json)
+`f0da2be968ef37abd37e5a639743006e41819e0c15ac19c3b99030352a8ea5ce` freezes
+44 files, five support changes/copies, unchanged M06 app `0857d612…66fa79a`, and
+test-only launcher APK `03df92d8…a178d30`. No product or existing JUnit scenario changed.
+
+[Recorded commands](../../evidence/phase-1/android-kotlin/M07/commands.jsonl): eight invocations;
+fixture 10/10 PASS (five old tests unchanged), runner-only build PASS, static checks PASS.
+Port checks returned expected exit 1 with no listener; other recorded commands exited 0.
+[One actual baseline](../../evidence/phase-1/android-kotlin/M07/baseline-android-01/result.json):
+111.428 s, 257 adb commands, A/B/C PASS at reproducing the limitation; no retry.
+A: PATCH 200 overwrites newer title at v3; PID 13073 → absent → 13690 (90 commands).
+B: DELETE 200 removes newer Item, tombstone v3; PID 13860 → absent → 14279 (79).
+C: PATCH 404 leaves optimistic Item/intent pending; PID 14457 → absent → 14951 (88).
+Each offline UI operation preceded the completed remote change and its one foreground Sync.
+Twelve raw native DB/WAL snapshots retain schema 4 and identical pre/post-reopen state.
+Three parked controllers intentionally terminated with their processes; no JUnit pass claimed.
+
+[Cleanup/byte audit](../../evidence/phase-1/android-kotlin/M07/baseline-byte-native-audit.json):
+fixture 81947 stopped by SIGINT, exit 0, PID absent/port 18080 free; app absent and
+network 0/1/1 with active network 115. All frozen bytes unchanged.
+Main independently accepted [raw evidence](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-baseline-audit.json>)
+and [cleanup](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M07/main-baseline-cleanup.json>).
+Final Android execution remains main-owned; no owner final device run is authorized.
diff --git a/verification/version_conflict_restart.py b/verification/version_conflict_restart.py
new file mode 100644
index 0000000..d84ca5e
--- /dev/null
+++ b/verification/version_conflict_restart.py
@@ -0,0 +1,314 @@
+#!/usr/bin/env python3
+"""Frozen M07 A/B/C actual UI conflicts and persistent reopen on the leased device.
+
+Reuse the M05 UI/network/native capture helpers. Only the initial test launcher
+injects fixed mutation IDs; each actual process restart uses plain MainActivity.
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
+INPUTS_PATH = Path(__file__).with_name("M07-inputs.json")
+INPUTS = json.loads(INPUTS_PATH.read_text())
+PATH = "/items/conflict-001"
+PENDING_COLUMNS = ["sequence", "itemId", "operation", "title", "completed",
+                   "clientMutationId", "payloadHash", "terminalError"]
+
+
+def envelope(case, fixed, title=None, base=1):
+    payload = {"title": title or INPUTS["localTitle"]} if case["operation"] == "RENAME" else {}
+    if fixed:
+        payload["baseVersion"] = base
+    canonical = json.dumps({"method": case["method"], "path": PATH, "payload": payload or None},
+                           sort_keys=True, separators=(",", ":"), ensure_ascii=False)
+    return payload, canonical, hashlib.sha256(canonical.encode()).hexdigest()
+
+
+def attached_own_records(activities):
+    assert activities.startswith("ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)"), "Malformed activity dump"
+    assert re.search(r"^Display #\d+ \(activities from top to bottom\):", activities, re.M), "Missing attached display hierarchy"
+    return [line.strip() for line in activities.splitlines()
+            if re.match(r"\s*\*\s+(?:Task\{|Hist\s+#\d+:\s*ActivityRecord\{)", line)
+            and re.search(re.escape(PACKAGE) + r"(?=[/}\s])", line)]
+
+
+class VersionConflictScenario(OfflineQueueScenario):
+    def __init__(self, args, case, install):
+        super().__init__(args)
+        self.case, self.install = case, install
+        self.fixed = args.expect == "conflicts"
+        self.result.update(scenario="M07", case=case["name"],
+                           harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+                           inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+                           fixtureLog=str(args.fixture_log.resolve()), fixtureLogStartOffset=self.fixture_offset)
+
+    def pre_seed_teardown(self, label):
+        deadline = time.monotonic() + INPUTS["harness"]["preSeedTeardownWaitSeconds"]
+        while True:
+            pid = self.adb("shell", "pidof", PACKAGE, allow_failure=True)
+            attached = attached_own_records(self.adb("shell", "dumpsys", "activity", "activities"))
+            if not pid and not attached:
+                self.result.setdefault("preSeedTeardown", []).append({"label": label, "appAbsent": True,
+                                                                      "attachedRecords": [], "lastCommand": self.command_count})
+                return
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"Pre-seed teardown incomplete: pid={pid!r}, attached={attached}")
+            time.sleep(0.2)
+
+    def start_launcher(self):
+        command = [str(self.args.adb), "-s", SERIAL, "shell", "am", "instrument", "-w", "-r",
+                   "-e", "m07Case", self.case["name"], LAUNCHER]
+        self.command_count += 1
+        with (self.output / "commands.txt").open("a") as log:
+            log.write(f"{self.command_count:03d} {shlex.join(command)} [parked until external force-stop]\n")
+        self.launcher_files = [(self.output / "launcher.stdout").open("wb"), (self.output / "launcher.stderr").open("wb")]
+        self.launcher = subprocess.Popen(command, stdout=self.launcher_files[0], stderr=self.launcher_files[1])
+        self.result["initialLauncher"] = {"command": command, "hostPid": self.launcher.pid,
+                                          "purpose": "fixed constructors before seed only; not a JUnit suite"}
+        deadline = time.monotonic() + INPUTS["harness"]["uiWaitSeconds"]
+        while "ready-for-external-ui-controller" not in (self.output / "launcher.stdout").read_text():
+            if self.launcher.poll() is not None or time.monotonic() >= deadline:
+                raise AssertionError("Initial M07 launcher did not become ready")
+            time.sleep(0.2)
+
+    def capture(self, name):
+        rows = self.snapshot(name)
+        folder = self.output / name
+        with tempfile.TemporaryDirectory(prefix="mse-m07-snapshot-") as temporary:
+            local = Path(temporary)
+            for suffix in ("", "-wal"):
+                if (folder / (DB_NAME + suffix)).exists():
+                    shutil.copyfile(folder / (DB_NAME + suffix), local / (DB_NAME + suffix))
+            with sqlite3.connect(local / DB_NAME) as database:
+                def table(name, expected, order):
+                    columns = [row[1] for row in database.execute(f"PRAGMA table_info({name})")]
+                    assert columns == expected, (name, columns)
+                    return [dict(zip(columns, row)) for row in database.execute(f"SELECT * FROM {name} ORDER BY {order}")] if columns else []
+                pending = table("pending_mutations", PENDING_COLUMNS + (["baseVersion", "dispatched"] if self.fixed else []), "sequence")
+                acknowledged = table("acknowledged_mutations", ["clientMutationId", "payloadHash", "statusCode", "responseBody"], "clientMutationId")
+                tombstones = table("tombstones", ["id", "version", "updatedAt", "deleted"] if self.fixed else [], "id")
+                metadata = [list(row) for row in database.execute("SELECT id,lastSuccessfulRefreshAt FROM sync_metadata ORDER BY id")]
+        captured = {"items": rows, "pending": pending, "acknowledged": acknowledged,
+                    "tombstones": tombstones, "refreshMetadata": metadata}
+        (folder / "storage.json").write_text(json.dumps(captured, indent=2) + "\n")
+        (self.output / f"{name}.png").write_bytes(self.adb("exec-out", "screencap", "-p", binary=True))
+        self.result.setdefault("snapshots", []).append(f"{name}/storage.json")
+        return captured
+
+    def original_pending(self, dispatched=False, conflict=False):
+        pending = {"sequence": 1, "itemId": INPUTS["seed"]["id"], "operation": self.case["operation"],
+                   "title": INPUTS["localTitle"] if self.case["operation"] == "RENAME" else None, "completed": None,
+                   "clientMutationId": self.case["clientMutationId"], "payloadHash": envelope(self.case, self.fixed)[2],
+                   "terminalError": "version_conflict" if conflict else None}
+        if self.fixed:
+            pending.update(baseVersion=1, dispatched=int(dispatched))
+        return pending
+
+    def rename(self, old, new):
+        self.tap(**{"content-desc": f"Edit {old}"})
+        self.tap(**{"class": "android.widget.EditText"})
+        self.adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        self.adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len(old)))
+        self.text(new)
+        self.tap(text="Save")
+        self.completed_ui([new])
+
+    def conflict_ui(self):
+        self.completed_ui([] if self.case["name"] == "C" else [INPUTS["canonicalItem"]["title"]])
+        self.wait_text(INPUTS["conflictUi"])
+
+    def assert_conflict(self, captured):
+        assert captured["pending"] == [self.original_pending(dispatched=True, conflict=True)], captured
+        assert captured["items"] == ([] if self.case["name"] == "C" else [INPUTS["canonicalItem"]]), captured
+        assert captured["tombstones"] == ([INPUTS["canonicalTombstone"]] if self.case["name"] == "C" else []), captured
+        assert captured["acknowledged"] == [], captured
+
+    def run(self):
+        self.result["initialNetwork"] = self.wait_network(True)
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        self.pre_seed_teardown("before-setup")
+        if self.install:
+            self.adb("install", "-r", str(self.args.apk.resolve()))
+            self.adb("install", "-r", str(self.args.test_apk.resolve()))
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.pre_seed_teardown("after-clear-before-launch")
+        initial = self.http("/__m07/reset", {})
+        assert initial["items"] == [INPUTS["seed"]] and initial["tombstones"] == []
+        assert initial["mutationRequests"] == initial["applied"] == 0
+        assert initial["nextTimestamp"] == INPUTS["localTimestamp"]
+        self.start_launcher()
+        self.completed_ui([])
+        self.tap(text="Sync")
+        self.wait_text("Fresh local data")
+        self.completed_ui([INPUTS["seed"]["title"]])
+        seeded = self.capture("seeded")
+        assert seeded["items"] == [INPUTS["seed"]] and seeded["pending"] == [] and seeded["acknowledged"] == []
+        assert seeded["refreshMetadata"] == [[1, INPUTS["localTimestamp"]]]
+
+        self.go_offline()
+        if self.case["operation"] == "RENAME":
+            self.rename(INPUTS["seed"]["title"], INPUTS["localTitle"])
+        else:
+            self.tap(**{"content-desc": "Delete Initial"})
+            self.completed_ui([])
+        self.wait_text("Pending changes: 1")
+        queued = self.capture("queued-offline")
+        local_rows = [{**INPUTS["seed"], "title": INPUTS["localTitle"], "updatedAt": INPUTS["localTimestamp"]}] if self.case["operation"] == "RENAME" else []
+        assert queued["items"] == local_rows and queued["pending"] == [self.original_pending()], queued
+        assert queued["acknowledged"] == [] and queued["tombstones"] == []
+        remote = self.http("/__m07/remote", {"operation": self.case["remoteOperation"]})
+        assert remote["items"] == ([] if self.case["name"] == "C" else [INPUTS["canonicalItem"]]), remote
+        assert remote["tombstones"] == ([INPUTS["canonicalTombstone"]] if self.case["name"] == "C" else []), remote
+        assert remote["mutationRequests"] == remote["applied"] == 0 and remote["remoteMutations"] == 1
+        self.result["offlineAfterRemoteChange"] = self.wait_network(False)
+        self.result["restoredNetwork"] = self.restore_network()
+        self.tap(text="Sync")
+        if self.fixed:
+            self.conflict_ui()
+        elif self.case["name"] == "C":
+            self.wait_text("Stale local data · sync error")
+            self.completed_ui([INPUTS["localTitle"]])
+        else:
+            self.wait_text("Fresh local data")
+            self.completed_ui([INPUTS["localTitle"]] if self.case["name"] == "A" else [])
+        before = self.capture("before-stop")
+        if self.fixed:
+            self.assert_conflict(before)
+        elif self.case["name"] == "C":
+            assert before == queued, before
+        else:
+            expected_rows = [{**INPUTS["seed"], "title": INPUTS["localTitle"], "version": 3, "updatedAt": 1700000501000}] if self.case["name"] == "A" else []
+            assert before["items"] == expected_rows and before["pending"] == [] and len(before["acknowledged"]) == 1, before
+
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit() and self.launcher.poll() is None, old_pid
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True), "Old process remains"
+        self.result["processAbsentAfterForceStop"] = True
+        self.record_launcher_termination(expected=True)
+        # No install, clear, native write, seed, or instrumentation across this boundary.
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        self.completed_ui([row["title"] for row in before["items"]])
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and new_pid != old_pid, (old_pid, new_pid)
+        self.result.update(beforePid=int(old_pid), afterPid=int(new_pid))
+        after = self.capture("after-relaunch")
+        assert after == before, "Durable state changed across the real process boundary"
+        assert self.http("/__m07")["mutationRequests"] == 1, "Restart resent an intent"
+
+        if self.fixed:
+            self.conflict_ui()
+            self.tap(text="Sync")  # Ordinary drain must exclude the retained conflict.
+            self.wait_text("Fresh local data")
+            self.conflict_ui()
+            self.assert_conflict(self.capture("after-ordinary-drain"))
+            assert self.http("/__m07")["mutationRequests"] == 1, "Ordinary drain resent a conflict"
+            if self.case["name"] == "A":
+                self.rename(INPUTS["canonicalItem"]["title"], INPUTS["explicitItem"]["title"])
+                fresh = self.capture("fresh-explicit-intent")
+                assert len(fresh["pending"]) == 2 and fresh["pending"][0] == before["pending"][0], fresh
+                new = fresh["pending"][1]
+                assert new["clientMutationId"] and new["clientMutationId"] != self.case["clientMutationId"]
+                assert new == {**self.original_pending(), "sequence": 2, "title": INPUTS["explicitItem"]["title"],
+                               "clientMutationId": new["clientMutationId"], "baseVersion": 2,
+                               "payloadHash": envelope(self.case, True, INPUTS["explicitItem"]["title"], 2)[2]}, new
+                self.tap(text="Sync")
+                self.wait_text("Fresh local data")
+                self.completed_ui([INPUTS["explicitItem"]["title"]])
+                final = self.capture("after-explicit-edit")
+                assert final["items"] == [INPUTS["explicitItem"]] and final["pending"] == before["pending"], final
+                assert final["tombstones"] == [] and len(final["acknowledged"]) == 1, final
+                receipt = final["acknowledged"][0]
+                assert receipt["clientMutationId"] == new["clientMutationId"] and receipt["payloadHash"] == new["payloadHash"]
+                assert receipt["statusCode"] == 200 and json.loads(receipt["responseBody"]) == {"item": INPUTS["explicitItem"]}
+                self.wait_text(INPUTS["conflictUi"])
+                self.result["freshIdentity"] = new["clientMutationId"]
+
+        remote = self.http("/__m07")
+        events = [json.loads(line) for line in self.args.fixture_log.read_bytes()[self.fixture_offset:].splitlines() if line.startswith(b"{")]
+        events = [event for event in events if event["method"] in ("POST", "PATCH", "DELETE") and event["path"].startswith("/items")]
+        count = 2 if self.fixed and self.case["name"] == "A" else 1
+        assert len(events) == remote["mutationRequests"] == count, events
+        payload, canonical, digest = envelope(self.case, self.fixed)
+        expected_request = {**payload, "clientMutationId": self.case["clientMutationId"], "payloadHash": digest}
+        first = events[0]
+        assert (first["method"], first["path"], first["canonical"], first["request"]) == (self.case["method"], PATH, canonical, expected_request), first
+        assert first["status"] == (409 if self.fixed else self.case["baselineStatus"]), first
+        if self.fixed:
+            assert first["response"] == {"error": "version_conflict", "item": None if self.case["name"] == "C" else INPUTS["canonicalItem"],
+                                          "tombstone": INPUTS["canonicalTombstone"] if self.case["name"] == "C" else None}, first
+            assert remote["versionConflicts"] == 1 and remote["applied"] == count - 1
+            expected_items = [] if self.case["name"] == "C" else [INPUTS["explicitItem"] if self.case["name"] == "A" else INPUTS["canonicalItem"]]
+            expected_tombstones = [INPUTS["canonicalTombstone"]] if self.case["name"] == "C" else []
+            if count == 2:
+                assert events[1]["status"] == 200 and events[1]["request"]["baseVersion"] == 2
+                assert events[1]["request"]["clientMutationId"] == self.result["freshIdentity"]
+        else:
+            assert remote["versionConflicts"] == 0 and remote["applied"] == (0 if self.case["name"] == "C" else 1)
+            expected_items = before["items"] if self.case["name"] != "C" else []
+            expected_tombstones = [{**INPUTS["canonicalTombstone"], "version": 3, "updatedAt": 1700000501000}] if self.case["name"] == "B" else ([INPUTS["canonicalTombstone"]] if self.case["name"] == "C" else [])
+        assert remote["items"] == expected_items and remote["tombstones"] == expected_tombstones, remote
+        assert remote["duplicates"] == remote["identityConflicts"] == remote["hashRejections"] == 0
+        assert remote["nextTimestamp"] == (1700000502000 if remote["applied"] else 1700000501000)
+        self.result.update(status="PASS", mutationHttpStatuses=[event["status"] for event in events],
+                           remoteCounts={key: remote[key] for key in ("applied", "mutationRequests", "versionConflicts", "duplicates", "remoteMutations")},
+                           fixtureLogEndOffset=self.args.fixture_log.stat().st_size)
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("limitation", "conflicts"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--test-apk", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--fixture-log", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, required=True)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    args = parser.parse_args()
+    output = args.output.resolve()
+    output.mkdir(parents=True, exist_ok=False)
+    results = []
+    try:
+        for index, case in enumerate(INPUTS["cases"]):
+            case_args = argparse.Namespace(**{**vars(args), "output": output / case["name"]})
+            scenario = VersionConflictScenario(case_args, case, install=index == 0)
+            try:
+                scenario.run()
+            except Exception as error:
+                scenario.result.update(status="FAIL", error=repr(error))
+                raise
+            finally:
+                try:
+                    scenario.cleanup()
+                except Exception as error:
+                    scenario.result.update(status="FAIL", cleanupError=repr(error))
+                    raise
+                finally:
+                    scenario.result["adbCommands"] = scenario.command_count
+                    (scenario.output / "result.json").write_text(json.dumps(scenario.result, indent=2) + "\n")
+                    results.append(scenario.result)
+    finally:
+        summary = {"scenario": "M07", "expectation": args.expect,
+                   "status": "PASS" if len(results) == 3 and all(case["status"] == "PASS" for case in results) else "FAIL",
+                   "cases": [{key: case.get(key) for key in ("case", "status", "adbCommands", "beforePid", "afterPid", "error", "mutationHttpStatuses", "cleanup")} for case in results],
+                   "adbCommands": sum(case["adbCommands"] for case in results)}
+        (output / "result.json").write_text(json.dumps(summary, indent=2) + "\n")
+        print(json.dumps(summary, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()


