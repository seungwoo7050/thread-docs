# M09 — Process Death 이후 복구

## `test(M09): preserve actual process-death baseline`

diff --git a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
index 1546f03..f31af7b 100644
--- a/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
+++ b/app/src/androidTest/java/com/mobilesystemsevolution/kotlin/M05ScenarioInstrumentation.kt
@@ -15,11 +15,13 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
     private var externalController = false
     private var m06 = false
     private var m07Case: String? = null
+    private var m09Case: String? = null
 
     override fun onCreate(arguments: Bundle?) {
         m06 = arguments?.getString("m06ExternalController") == "true"
         m07Case = arguments?.getString("m07Case")?.also { require(it in setOf("A", "B", "C")) }
-        externalController = m07Case != null || m06 || arguments?.getString("m05ExternalController") == "true"
+        m09Case = arguments?.getString("m09Case")?.also { require(it in setOf("A", "B")) }
+        externalController = m09Case != null || m07Case != null || m06 || arguments?.getString("m05ExternalController") == "true"
         super.onCreate(arguments)
     }
 
@@ -36,18 +38,23 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
             val opened = ItemDatabase.open(targetContext)
             database = opened
             val initialTime = when {
+                m09Case != null -> 1_700_000_600_000L
                 m07Case != null -> 1_700_000_500_000L
                 m06 -> 1_700_000_400_000L
                 else -> 1_700_000_300_000L
             }
             var localTime = initialTime
-            val store = ItemStore(opened.items(), nextId = { if (m06) "crash-001" else "device-001" },
+            val store = ItemStore(opened.items(), nextId = {
+                if (m09Case != null) "death-001" else if (m06) "crash-001" else "device-001"
+            },
                 now = { localTime.also { localTime += 1_000 } },
                 nextMutationId = {
-                    when (m07Case) {
-                        "A" -> "m07-update-001"
-                        "B" -> "m07-delete-001"
-                        "C" -> "m07-deleted-001"
+                    when {
+                        m09Case == "A" -> "m09-create-001"
+                        m09Case == "B" -> "m09-update-001"
+                        m07Case == "A" -> "m07-update-001"
+                        m07Case == "B" -> "m07-delete-001"
+                        m07Case == "C" -> "m07-deleted-001"
                         else -> if (m06) "m06-create-001" else UUID.randomUUID().toString()
                     }
                 })
@@ -55,7 +62,7 @@ class M05ScenarioInstrumentation : AndroidJUnitRunner() {
             runOnMainSync { activity.setContent { MaterialTheme { ItemScreen(store, sync) } } }
             waitForIdleSync()
             sendStatus(1, Bundle().apply {
-                putString(if (m07Case != null) "m07" else if (m06) "m06" else "m05", "ready-for-external-ui-controller")
+                putString(if (m09Case != null) "m09" else if (m07Case != null) "m07" else if (m06) "m06" else "m05", "ready-for-external-ui-controller")
             })
             // This is not a JUnit test. The required force-stop intentionally terminates it.
             CountDownLatch(1).await(300, TimeUnit.SECONDS)
diff --git a/fixture/server.py b/fixture/server.py
index 99f4262..14c0fd3 100644
--- a/fixture/server.py
+++ b/fixture/server.py
@@ -1,11 +1,16 @@
 #!/usr/bin/env python3
-"""Local Item fixture with the fixed M06 replay and M07 version-conflict contracts."""
+"""Local Item fixture with fixed replay, version-conflict and held-response controls."""
 
 import argparse
+from functools import wraps
 import hashlib
-from http.server import BaseHTTPRequestHandler, HTTPServer
+from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
 import json
 import re
+import select
+import socket
+from threading import Event, RLock
+import time
 from urllib.parse import unquote
 
 
@@ -18,8 +23,30 @@ def payload_hash(method, path, payload):
     return hashlib.sha256(canonical_request(method, path, payload).encode("utf-8")).hexdigest()
 
 
-class Fixture(HTTPServer):
+def serialized_response(action):
+    """Keep canonical decisions serial; only response I/O and the M09 hold run unlocked."""
+    @wraps(action)
+    def handle(self):
+        self.held_response = None
+        self.response_headers_sent = False
+        with self.server.state_lock:
+            action(self)
+            status, payload, delivery = self.pending_response
+            held = self.held_response
+        if held is not None:
+            released = held["release"].wait(max(0, held["deadline"] - time.monotonic()))
+            with self.server.state_lock:
+                held["outcome"] = "dropped" if released else "expired"
+                held["finished"] = time.monotonic()
+                self.server.dropped_responses += 1
+            delivery = "dropped"
+        self.write_response(status, payload, delivery)
+    return handle
+
+
+class Fixture(ThreadingHTTPServer):
     def __init__(self, address):
+        self.state_lock = RLock()
         super().__init__(address, Handler)
         self.reset()
 
@@ -46,6 +73,9 @@ class Fixture(HTTPServer):
         self.version_conflicts = 0
         self.remote_mutations = 0
         self.requests = []
+        self.m09_case = None
+        self.hold_identity = None
+        self.held = None
 
     def tick(self):
         timestamp = self.next_timestamp
@@ -88,10 +118,44 @@ class Fixture(HTTPServer):
         self.tombstones[item_id] = dict(id=item_id, version=item["version"] + 1,
                                        updatedAt=self.tick(), deleted=True)
 
+    def hold_state(self):
+        if self.held is None:
+            return None
+        held = self.held
+        connection = held["handler"].connection
+        observed = time.monotonic()
+        try:
+            readable, _, _ = select.select([connection], [], [], 0)
+            opened = not readable or bool(connection.recv(1, socket.MSG_PEEK))
+            probe_error = None
+        except (OSError, ValueError) as error:
+            opened, probe_error = False, type(error).__name__
+        return dict(clientMutationId=held["identity"], outcome=held["outcome"],
+                    committed=True, maximumHoldMs=30000,
+                    committedMonotonic=held["started"], deadlineMonotonic=held["deadline"],
+                    observedMonotonic=observed, elapsedMs=round((observed - held["started"]) * 1000, 3),
+                    connectionOpen=opened, connectionProbeError=probe_error,
+                    headersSent=held["handler"].response_headers_sent,
+                    releaseRequested=held["release"].is_set(),
+                    finishedMonotonic=held.get("finished"), request=held["request"])
+
+    def death_state(self):
+        return dict(self.conflict_state(), case=self.m09_case, barrier=self.hold_state())
+
 
 class Handler(BaseHTTPRequestHandler):
     def respond(self, status, payload, delivery="delivered"):
+        # Freeze mutable canonical rows under the state lock, before releasing it for I/O.
+        self.pending_response = status, json.loads(json.dumps(payload)), delivery
+
+    def write_response(self, status, payload, delivery):
         body = json.dumps(payload, separators=(",", ":"), ensure_ascii=False).encode("utf-8")
+        evidence = {"method": self.command, "path": self.path, "status": status,
+                    "response": payload, "delivery": delivery,
+                    **getattr(self, "mutation_evidence", {})}
+        if hasattr(self, "mutation_evidence"):
+            with self.server.state_lock:
+                self.server.requests.append(json.loads(json.dumps(evidence)))
         if delivery == "dropped":
             # Commit/result recording already happened; return without writing HTTP bytes.
             self.close_connection = True
@@ -100,12 +164,8 @@ class Handler(BaseHTTPRequestHandler):
             self.send_header("Content-Type", "application/json")
             self.send_header("Content-Length", str(len(body)))
             self.end_headers()
+            self.response_headers_sent = True
             self.wfile.write(body)
-        evidence = {"method": self.command, "path": self.path, "status": status,
-                    "response": payload, "delivery": delivery,
-                    **getattr(self, "mutation_evidence", {})}
-        if hasattr(self, "mutation_evidence"):
-            self.server.requests.append(json.loads(json.dumps(evidence)))
         print(json.dumps(evidence), flush=True)
 
     def log_message(self, *_args):
@@ -172,6 +232,19 @@ class Handler(BaseHTTPRequestHandler):
     def applied_response(self, status, payload):
         frozen = self.cache_response(status, payload)
         self.server.applied += 1
+        if self.mutation_identity is not None and self.mutation_identity == self.server.hold_identity:
+            started = time.monotonic()
+            self.server.hold_identity = None
+            self.held_response = self.server.held = dict(
+                identity=self.mutation_identity, handler=self, release=Event(),
+                started=started, deadline=started + 30, outcome="held",
+                request=json.loads(json.dumps(self.mutation_evidence)),
+            )
+            print(json.dumps(dict(event="m09-committed-response-held", method=self.command,
+                                  path=self.path, status=status, response=frozen,
+                                  barrier=self.server.hold_state())), flush=True)
+            self.respond(status, frozen)
+            return
         drop = self.command == "POST" and payload.get("item", {}).get("id") == self.server.drop_create_id
         if drop:
             self.server.drop_create_id = None
@@ -192,6 +265,7 @@ class Handler(BaseHTTPRequestHandler):
         self.respond(409, response)
         return True
 
+    @serialized_response
     def do_GET(self):
         if self.path == "/items":
             self.server.get_requests += 1
@@ -208,10 +282,49 @@ class Handler(BaseHTTPRequestHandler):
             self.respond(200, self.server.mutation_state())
         elif self.path == "/__m07":
             self.respond(200, self.server.conflict_state())
+        elif self.path == "/__m09":
+            self.respond(200, self.server.death_state())
         else:
             self.respond(404, {"error": "not found"})
 
+    @serialized_response
     def do_POST(self):
+        if self.path in ("/__reset", "/__m06/reset", "/__m07/reset", "/__m09/reset"):
+            if self.server.held is not None and self.server.held["outcome"] == "held":
+                self.respond(409, {"error": "held_response_must_finish_before_reset"})
+                return
+        if self.path == "/__m09/reset":
+            try:
+                data = self.payload()
+                if set(data) != {"case"} or data["case"] not in ("A", "B"):
+                    raise ValueError("Expected case A or B")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            self.server.reset()
+            self.server.m09_case = data["case"]
+            self.server.items = {} if data["case"] == "A" else {
+                "death-001": dict(id="death-001", title="Initial", completed=False,
+                                  version=1, updatedAt=1700000000000),
+            }
+            self.server.next_timestamp = 1700000600000
+            self.server.hold_identity = "m09-update-001" if data["case"] == "B" else None
+            self.respond(200, self.server.death_state())
+            return
+        if self.path == "/__m09/release":
+            try:
+                if self.payload() != {"action": "drop"}:
+                    raise ValueError("Expected action drop")
+            except (ValueError, TypeError) as error:
+                self.respond(400, {"error": str(error)})
+                return
+            held = self.server.held
+            if held is None or held["outcome"] != "held" or held["release"].is_set():
+                self.respond(409, {"error": "no_unreleased_held_response"})
+                return
+            held["release"].set()
+            self.respond(200, self.server.death_state())
+            return
         if self.path == "/__m07/reset":
             try:
                 if self.payload():
@@ -310,6 +423,7 @@ class Handler(BaseHTTPRequestHandler):
         item_id = unquote(self.path[len("/items/"):])
         return item_id if item_id in self.server.items else None
 
+    @serialized_response
     def do_PATCH(self):
         try:
             data = self.mutation_payload()
@@ -332,6 +446,7 @@ class Handler(BaseHTTPRequestHandler):
         item.update(changes, version=item["version"] + 1, updatedAt=self.server.tick())
         self.applied_response(200, {"item": item})
 
+    @serialized_response
     def do_DELETE(self):
         try:
             data = self.mutation_payload()
diff --git a/fixture/test_server.py b/fixture/test_server.py
index eb4c3b3..aa0433d 100644
--- a/fixture/test_server.py
+++ b/fixture/test_server.py
@@ -2,6 +2,7 @@ import json
 from contextlib import contextmanager
 from http.client import RemoteDisconnected
 from threading import Thread
+import time
 import unittest
 from urllib.request import Request, urlopen
 from urllib.error import HTTPError
@@ -38,6 +39,86 @@ def mutation_wire(method, path, payload, identity):
 
 
 class FixtureContractTest(unittest.TestCase):
+    def test_m09_pending_create_fixed_contract(self):
+        payload = dict(id="death-001", title="Recovered create", completed=False)
+        wire = mutation_wire("POST", "/items", payload, "m09-create-001")
+        self.assertEqual("4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4", wire["payloadHash"])
+        with m06_fixture() as request:
+            state = request("POST", "/__m09/reset", {"case": "A"})[1]
+            self.assertEqual(([], 0, 0, 0, 1700000600000, None),
+                             (state["items"], state["applied"], state["duplicates"],
+                              state["mutationRequests"], state["nextTimestamp"], state["barrier"]))
+            item = dict(payload, version=1, updatedAt=1700000600000)
+            self.assertEqual((201, {"item": item}), request("POST", "/items", wire)[:2])
+            state = request("GET", "/__m09")[1]
+            self.assertEqual(([item], 1, 0, 1, 1700000601000),
+                             (state["items"], state["applied"], state["duplicates"],
+                              state["mutationRequests"], state["nextTimestamp"]))
+            self.assertEqual(wire, state["requests"][0]["request"])
+
+    def test_m09_committed_headerless_response_keeps_controls_live_then_exact_replay(self):
+        path = "/items/death-001"
+        wire = mutation_wire("PATCH", path, {"title": "Recovered update", "baseVersion": 1}, "m09-update-001")
+        self.assertEqual("53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12", wire["payloadHash"])
+        item = dict(id="death-001", title="Recovered update", completed=False,
+                    version=2, updatedAt=1700000600000)
+        with m06_fixture() as request:
+            request("POST", "/__m09/reset", {"case": "B"})
+            result = {}
+
+            def send():
+                try:
+                    result["response"] = request("PATCH", path, wire)
+                except Exception as error:
+                    result["error"] = type(error).__name__
+
+            sender = Thread(target=send, daemon=True)
+            sender.start()
+            try:
+                deadline = time.monotonic() + 5
+                while True:
+                    state = request("GET", "/__m09")[1]
+                    if state["barrier"] is not None:
+                        break
+                    self.assertLess(time.monotonic(), deadline, "No committed hold")
+                    time.sleep(0.01)
+                held = state["barrier"]
+                self.assertEqual(([item], 1, 0, 1, 0),
+                                 (state["items"], state["applied"], state["duplicates"],
+                                  state["mutationRequests"], state["droppedResponses"]))
+                self.assertEqual(("held", True, False, True, 30000),
+                                 (held["outcome"], held["committed"], held["headersSent"],
+                                  held["connectionOpen"], held["maximumHoldMs"]))
+                self.assertEqual(wire, held["request"]["request"])
+                self.assertEqual(30, held["deadlineMonotonic"] - held["committedMonotonic"])
+                self.assertEqual({}, result)
+                self.assertTrue(sender.is_alive())
+                # A second control round trip must complete while the original socket is open.
+                self.assertEqual(200, request("GET", "/__control")[0])
+                fresh = request("GET", "/__m09")[1]["barrier"]
+                self.assertGreater(fresh["observedMonotonic"], held["observedMonotonic"])
+                self.assertTrue(fresh["connectionOpen"])
+                self.assertFalse(fresh["headersSent"])
+                self.assertEqual(200, request("POST", "/__m09/release", {"action": "drop"})[0])
+                sender.join(timeout=5)
+                self.assertFalse(sender.is_alive())
+                self.assertEqual({"error": "RemoteDisconnected"}, result)
+                self.assertEqual((200, {"item": item}), request("PATCH", path, wire)[:2])
+                state = request("GET", "/__m09")[1]
+                self.assertEqual(([item], 1, 1, 2, 1, 1700000601000),
+                                 (state["items"], state["applied"], state["duplicates"],
+                                  state["mutationRequests"], state["droppedResponses"], state["nextTimestamp"]))
+                self.assertEqual(["dropped", "replayed"], [row["delivery"] for row in state["requests"]])
+                self.assertEqual([wire, wire], [row["request"] for row in state["requests"]])
+                self.assertEqual("dropped", state["barrier"]["outcome"])
+                self.assertFalse(state["barrier"]["headersSent"])
+            finally:
+                if sender.is_alive():
+                    barrier = request("GET", "/__m09")[1]["barrier"]
+                    if barrier is not None and barrier["outcome"] == "held" and not barrier["releaseRequested"]:
+                        request("POST", "/__m09/release", {"action": "drop"})
+                    sender.join(timeout=5)
+
     def test_m07_fixed_conflicts_and_fresh_explicit_edit(self):
         path = "/items/conflict-001"
         seed = dict(id="conflict-001", title="Initial", completed=False,
diff --git a/verification/M09-inputs.json b/verification/M09-inputs.json
new file mode 100644
index 0000000..da909ae
--- /dev/null
+++ b/verification/M09-inputs.json
@@ -0,0 +1,23 @@
+{
+  "thread": "M09",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "8f39fb5cb565d04035c3623da56b8326cb03171d",
+  "itemId": "death-001",
+  "localTimestamp": 1700000600000,
+  "acceptedTimestamp": 1700000600000,
+  "seedB": {"id":"death-001","title":"Initial","completed":false,"version":1,"updatedAt":1700000000000},
+  "cases": [
+    {"name":"A","operation":"CREATE","method":"POST","path":"/items","title":"Recovered create","clientMutationId":"m09-create-001","payload":{"id":"death-001","title":"Recovered create","completed":false},"payloadHash":"4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4","localVersion":0,"canonicalVersion":1,"statusCode":201},
+    {"name":"B","operation":"RENAME","method":"PATCH","path":"/items/death-001","title":"Recovered update","clientMutationId":"m09-update-001","payload":{"title":"Recovered update","baseVersion":1},"payloadHash":"53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12","localVersion":1,"canonicalVersion":2,"statusCode":200}
+  ],
+  "transport": {"fixturePort":18080,"httpCallTimeoutSeconds":10,"maximumResponseHoldMs":30000},
+  "harness": {"uiWaitSeconds":30,"adbCommandTimeoutSeconds":45,"networkWaitSeconds":30,"preSeedTeardownWaitSeconds":30,"initialLauncherTimeoutSeconds":300,"launcherTerminationWaitSeconds":15},
+  "baseline": "The exact M08 app reopens durable pending intent in STALE state without automatic HTTP. A retains an unsent creation and the fixture stays empty. B retains the original dispatched rename, remote applied1/duplicate0, and no client ACK. Observe actual rendered ready state, native storage, and unchanged fixture counters after ordinary cold startup; do not tap Sync after restart.",
+  "fixed": "Ordinary online cold startup alone drains the original stored intent. A finishes one canonical v1 Item, applied1/duplicate0/pending0. B replays its immutable base1 identity/hash after a genuinely in-flight kill, yielding canonical v2, applied1/duplicate1/pending0 and a stored ACK.",
+  "initialLauncher": "Test-only fixed ID/clock constructors before the process boundary, using actual ItemScreen/ItemStore. No JUnit pass is claimed when external force-stop terminates this parked controller. No injection, install, clear or mutation reconstruction after the boundary.",
+  "caseAOrder": "Reset empty; disable both data paths before initial launch; create through actual UI; prove committed native Item+intent and no HTTP; kill and prove absence while offline; read stopped native storage; restore online and launch normal MainActivity once.",
+  "caseBOrder": "Reset and arm headerless barrier; initial actual UI Sync downloads seed; actual UI rename commits base1 intent; capture native pending and PID before dispatch. Tap Sync once, observe committed hold, capture only native in-flight state, then fresh socket-open/headerless control immediately before force-stop. Prove PID absent within the unchanged 10-second client deadline; only then drop held response and read stopped storage. Launch normal MainActivity once without reset/reseed/manual Sync.",
+  "barrierProof": "The fixture serializes canonical state but releases that lock for response I/O and the maximum-30-second Event wait. GET /__m09 actively probes the held socket with select/peek and reports header state and monotonic observation time; no UI dump or screenshot occurs after committed hold before kill. An expired/closed-before-kill response fails the case.",
+  "scopeLimits": "M09 foreground process recovery only. No background scheduler, push, extra retry, new mutation semantics, M10 or phase-2 work. Existing helper gates and timeouts remain unchanged."
+}
diff --git a/verification/M09.md b/verification/M09.md
new file mode 100644
index 0000000..6e35210
--- /dev/null
+++ b/verification/M09.md
@@ -0,0 +1,49 @@
+# M09 — phase-1 process-death startup recovery
+
+Status: unchanged-M08 limitation baseline independently accepted. Product implementation
+and final M09 verification are pending; this is not a completion claim.
+START `8f39fb5cb565d04035c3623da56b8326cb03171d`; SPEC_REVISION
+`61280dd86ce88b6e431f408241c0998a275960aa`; attempt1, repair0/2.
+
+## Frozen baseline
+
+Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M09`.
+[Manifest](../../evidence/phase-1/android-kotlin/M09/baseline-frozen-01/manifest.json)
+`9c10d27f437211007f694b3c67fb9aa0684a527a51149a92e47d432cd3d80914` freezes 53 source
+hashes and five support copies. The app remains exact verified M08 `05b81312…a5dea`;
+test APK `16a5e396…35b0` adds only initial fixed constructors for the external controller.
+The fixture adds the bounded headerless response/control support; canonical decisions stay
+serialized, with the lock released for response I/O and the held wait. No product change.
+
+[Commands](../../evidence/phase-1/android-kotlin/M09/commands.jsonl): eight baseline invocations;
+seven exit0 and the expected pre-start free-port check exit1/empty. Fixture12/12 passed,
+including all ten unchanged prior test methods. The test-only build passed; no app build
+or product host-suite rerun. Static checks verified all 48 unaffected START files, existing
+21 public-void JUnit methods, controller bytecode, actual test DEX and runner registration.
+No failure or retry occurred.
+
+Main executed the frozen [combined baseline](../../evidence/phase-1/android-kotlin/M09/baseline-android-01/result.json)
+once: wrapper65.766 s, 150 adb commands, both limitation cases reproduced. The two parked
+instrumentation controllers were intentionally terminated with the actual app processes;
+neither is reported as a passing JUnit suite. Cold relaunch used ordinary MainActivity,
+with no post-restart Sync, install, clear, injection or reconstructed mutation.
+
+| Case | Actual PID boundary | Reproduced limitation |
+| --- | --- | --- |
+| A | 27330 → absent → 27623 | Offline committed create retained identity m09-create-001 and pending1; online cold startup sent nothing, remote empty/applied0/duplicate0. |
+| B | 27845 → absent → 28183 | Original m09-update-001/base1/hash/dispatched1 survived; remote v2 was committed, but startup left pending1/ACK absent with applied1/duplicate0. |
+
+B's fresh socket-open/headerless observation was at monotonic144158.62177075, followed by
+confirmed PID absence at144158.80144275 within the unchanged 10-second client deadline.
+The response was dropped only after absence; the maximum 30000 ms barrier stayed unchanged.
+No UI dump or screenshot consumed the committed-response window.
+
+[Main audit](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-baseline-audit.json>)
+directly checked nine native SQLite/WAL snapshots, all original identities/hashes, actual
+PID/HTTP/UI evidence and 366 raw artifact hashes; all 53 source hashes and both APKs stayed
+unchanged. [Cleanup](</Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M09/main-baseline-cleanup.json>):
+owned fixture90586 stopped with SIGINT and exited0; direct checks proved PID absent/18080
+free, app absent, network0/1/1 active131. Main released the lease. No owner device run.
+
+This commit preserves baseline support/evidence only. TRACK remains at M08; implementation
+requires main's separate authorization after its Git/history audit.
diff --git a/verification/startup_recovery.py b/verification/startup_recovery.py
new file mode 100644
index 0000000..2348649
--- /dev/null
+++ b/verification/startup_recovery.py
@@ -0,0 +1,333 @@
+#!/usr/bin/env python3
+"""Frozen M09 actual process death with pending work and a committed, held response.
+
+Requires an exclusive device lease and owned fixture. Only the initial launcher
+injects fixed constructors; every cold relaunch is ordinary production MainActivity.
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
+from version_conflict_restart import VersionConflictScenario
+
+
+INPUTS_PATH = Path(__file__).with_name("M09-inputs.json")
+INPUTS = json.loads(INPUTS_PATH.read_text())
+PENDING_COLUMNS = ["sequence", "itemId", "operation", "title", "completed", "clientMutationId",
+                   "payloadHash", "terminalError", "baseVersion", "dispatched"]
+
+
+class StartupRecoveryScenario(OfflineQueueScenario):
+    pre_seed_teardown = VersionConflictScenario.pre_seed_teardown
+    rename = VersionConflictScenario.rename
+
+    def __init__(self, args, case, install):
+        super().__init__(args)
+        self.case, self.install = case, install
+        self.fixed = args.expect == "recovered"
+        self.result.update(scenario="M09", case=case["name"],
+                           harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+                           inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+                           fixtureLog=str(args.fixture_log.resolve()), fixtureLogStartOffset=self.fixture_offset,
+                           manualSyncAfterRestart=0, nativeWritesAfterRestart=0)
+
+    def adb(self, *args, **kwargs):
+        started = time.monotonic()
+        number = self.command_count + 1
+        try:
+            return super().adb(*args, **kwargs)
+        finally:
+            with (self.output / "command-times.jsonl").open("a") as log:
+                log.write(json.dumps(dict(command=number, startedMonotonic=started,
+                                          endedMonotonic=time.monotonic())) + "\n")
+
+    def start_launcher(self):
+        command = [str(self.args.adb), "-s", SERIAL, "shell", "am", "instrument", "-w", "-r",
+                   "-e", "m09Case", self.case["name"], LAUNCHER]
+        self.command_count += 1
+        with (self.output / "commands.txt").open("a") as log:
+            log.write(f"{self.command_count:03d} {shlex.join(command)} [parked until external force-stop]\n")
+        self.launcher_files = [(self.output / "launcher.stdout").open("wb"), (self.output / "launcher.stderr").open("wb")]
+        self.launcher = subprocess.Popen(command, stdout=self.launcher_files[0], stderr=self.launcher_files[1])
+        self.result["initialLauncher"] = dict(command=command, hostPid=self.launcher.pid,
+                                             purpose="fixed constructors before death only; not a JUnit suite")
+        deadline = time.monotonic() + INPUTS["harness"]["uiWaitSeconds"]
+        while "ready-for-external-ui-controller" not in (self.output / "launcher.stdout").read_text():
+            if self.launcher.poll() is not None or time.monotonic() >= deadline:
+                raise AssertionError("Initial M09 launcher did not become ready")
+            time.sleep(0.2)
+
+    def capture(self, name):
+        # Native reads only: the B in-flight window must not spend time dumping UI/images.
+        rows = self.snapshot(name)
+        folder = self.output / name
+        with tempfile.TemporaryDirectory(prefix="mse-m09-snapshot-") as temporary:
+            local = Path(temporary)
+            for suffix in ("", "-wal"):
+                if (folder / (DB_NAME + suffix)).exists():
+                    shutil.copyfile(folder / (DB_NAME + suffix), local / (DB_NAME + suffix))
+            with sqlite3.connect(local / DB_NAME) as database:
+                assert database.execute("PRAGMA integrity_check").fetchone() == ("ok",)
+
+                def table(name, expected, order):
+                    columns = [row[1] for row in database.execute(f"PRAGMA table_info({name})")]
+                    assert columns == expected, (name, columns)
+                    return [dict(zip(columns, row)) for row in database.execute(f"SELECT * FROM {name} ORDER BY {order}")]
+
+                pending = table("pending_mutations", PENDING_COLUMNS, "sequence")
+                acknowledged = table("acknowledged_mutations", ["clientMutationId", "payloadHash", "statusCode", "responseBody"], "clientMutationId")
+                tombstones = table("tombstones", ["id", "version", "updatedAt", "deleted"], "id")
+                metadata = [list(row) for row in database.execute("SELECT id,lastSuccessfulRefreshAt FROM sync_metadata ORDER BY id")]
+                allocator = [list(row) for row in database.execute("SELECT name,seq FROM sqlite_sequence ORDER BY name")]
+        captured = dict(items=rows, pending=pending, acknowledged=acknowledged,
+                        tombstones=tombstones, refreshMetadata=metadata, allocator=allocator)
+        (folder / "storage.json").write_text(json.dumps(captured, indent=2) + "\n")
+        self.result.setdefault("snapshots", []).append(f"{name}/storage.json")
+        return captured
+
+    def original_pending(self, dispatched=False):
+        case = self.case
+        return dict(sequence=1, itemId=INPUTS["itemId"], operation=case["operation"], title=case["title"],
+                    completed=0 if case["name"] == "A" else None,
+                    clientMutationId=case["clientMutationId"], payloadHash=case["payloadHash"],
+                    terminalError=None, baseVersion=None if case["name"] == "A" else 1,
+                    dispatched=int(dispatched))
+
+    def assert_pending(self, captured, dispatched=False):
+        case = self.case
+        assert captured == dict(
+            items=[dict(id=INPUTS["itemId"], title=case["title"], completed=0,
+                        version=case["localVersion"], updatedAt=INPUTS["localTimestamp"])],
+            pending=[self.original_pending(dispatched)], acknowledged=[], tombstones=[],
+            refreshMetadata=[] if case["name"] == "A" else [[1, INPUTS["localTimestamp"]]],
+            allocator=[["pending_mutations", 1]],
+        ), captured
+
+    def wait_fixture(self, predicate, description, timeout):
+        deadline = time.monotonic() + timeout
+        while True:
+            state = self.http("/__m09")
+            if predicate(state):
+                return state
+            if time.monotonic() >= deadline:
+                raise AssertionError(f"Fixture did not reach {description}: {state}")
+            time.sleep(0.05)
+
+    def dispatch_sync(self):
+        tree = self.wait_for(lambda value: bool(self.matching(value, text="Sync", enabled="true")), "enabled Sync")
+        nodes = self.matching(tree, text="Sync", enabled="true")
+        assert len(nodes) == 1
+        bounds = [int(value) for value in re.findall(r"\d+", nodes[0].get("bounds", ""))]
+        assert len(bounds) == 4
+        self.result["dispatchMonotonic"] = time.monotonic()
+        self.result["dispatchInputCommand"] = self.command_count + 1
+        self.adb("shell", "input", "tap", str((bounds[0] + bounds[2]) // 2), str((bounds[1] + bounds[3]) // 2))
+
+    def assert_held(self, state):
+        held = state["barrier"]
+        assert state["items"] == [self.canonical_item()]
+        assert (state["applied"], state["duplicates"], state["mutationRequests"], state["droppedResponses"]) == (1, 0, 1, 0)
+        assert held is not None and held["outcome"] == "held" and held["committed"]
+        assert held["connectionOpen"] and not held["headersSent"] and not held["releaseRequested"], held
+        assert held["connectionProbeError"] is None and held["maximumHoldMs"] == 30000
+        assert held["elapsedMs"] < 30000
+        assert held["clientMutationId"] == self.case["clientMutationId"]
+        assert held["request"]["request"] == self.wire()
+        assert held["request"]["canonical"] == self.canonical_request()
+        assert held["request"]["actualPayloadHash"] == self.case["payloadHash"]
+
+    def wire(self):
+        return dict(self.case["payload"], clientMutationId=self.case["clientMutationId"], payloadHash=self.case["payloadHash"])
+
+    def canonical_request(self):
+        return json.dumps(dict(method=self.case["method"], path=self.case["path"], payload=self.case["payload"]),
+                          sort_keys=True, separators=(",", ":"), ensure_ascii=False)
+
+    def canonical_item(self):
+        return dict(id=INPUTS["itemId"], title=self.case["title"], completed=False,
+                    version=self.case["canonicalVersion"], updatedAt=INPUTS["acceptedTimestamp"])
+
+    def run(self):
+        started = time.monotonic()
+        case = self.case
+        self.result["initialNetwork"] = self.wait_network(True)
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        self.pre_seed_teardown("before-setup")
+        if self.install:
+            self.adb("install", "-r", str(self.args.apk.resolve()))
+            self.adb("install", "-r", str(self.args.test_apk.resolve()))
+        assert self.adb("shell", "pm", "clear", PACKAGE) == "Success"
+        self.pre_seed_teardown("after-clear-before-launch")
+        initial = self.http("/__m09/reset", {"case": case["name"]})
+        assert initial["items"] == ([] if case["name"] == "A" else [INPUTS["seedB"]])
+        assert initial["mutationRequests"] == initial["applied"] == initial["duplicates"] == 0
+        assert initial["nextTimestamp"] == INPUTS["acceptedTimestamp"] and initial["barrier"] is None
+        if case["name"] == "A":
+            self.go_offline()
+        self.start_launcher()
+        self.completed_ui([])
+        if case["name"] == "B":
+            self.tap(text="Sync")  # Initial seed download, before the tested pending operation.
+            self.wait_text("Fresh local data")
+            self.completed_ui([INPUTS["seedB"]["title"]])
+        seeded = self.capture("seeded")
+        assert seeded["items"] == initial["items"] and seeded["pending"] == seeded["acknowledged"] == []
+        if case["name"] == "A":
+            self.tap(**{"class": "android.widget.EditText"})
+            self.text(case["title"])
+            self.tap(text="Add")
+            self.completed_ui([case["title"]])
+        else:
+            self.rename(INPUTS["seedB"]["title"], case["title"])
+        self.wait_text("Pending changes: 1")
+        self.assert_pending(self.capture("queued"))
+        before = self.http("/__m09")
+        assert before["mutationRequests"] == before["applied"] == 0
+        old_pid = self.adb("shell", "pidof", PACKAGE)
+        assert old_pid.isdigit() and self.launcher.poll() is None
+        self.result["beforePid"] = int(old_pid)
+        if case["name"] == "A":
+            self.result["networkBeforeDeath"] = self.wait_network(False)
+        else:
+            self.dispatch_sync()
+            held = self.wait_fixture(lambda value: value["barrier"] is not None, "committed hold", 10)
+            self.assert_held(held)
+            self.assert_pending(self.capture("in-flight"), dispatched=True)
+            self.result["inFlightCaptureEndedMonotonic"] = time.monotonic()
+            fresh = self.http("/__m09")  # The very next device command is the real kill.
+            self.assert_held(fresh)
+            assert fresh["barrier"]["observedMonotonic"] >= self.result["inFlightCaptureEndedMonotonic"]
+            self.result["freshHeldImmediatelyBeforeKill"] = fresh
+        self.result["forceStopCommand"] = self.command_count + 1
+        self.result["forceStopStartedMonotonic"] = time.monotonic()
+        self.adb("shell", "am", "force-stop", PACKAGE)
+        assert not self.adb("shell", "pidof", PACKAGE, allow_failure=True), "Old process remains"
+        self.result.update(processAbsentAfterForceStop=True, absentMonotonic=time.monotonic(),
+                           absenceCommand=self.command_count)
+        if case["name"] == "B":
+            assert self.result["absentMonotonic"] - self.result["dispatchMonotonic"] < 10, "Kill missed the unchanged client deadline"
+            self.result["dropRequestedAfterAbsenceMonotonic"] = time.monotonic()
+            self.http("/__m09/release", {"action": "drop"})
+            dropped = self.wait_fixture(lambda value: value["barrier"]["outcome"] == "dropped" and len(value["requests"]) == 1,
+                                        "completed drop", INPUTS["harness"]["uiWaitSeconds"])
+            assert dropped["droppedResponses"] == 1 and not dropped["barrier"]["headersSent"]
+        self.record_launcher_termination(expected=True)
+        stopped = self.capture("stopped")
+        self.assert_pending(stopped, dispatched=case["name"] == "B")
+        if case["name"] == "A":
+            self.result["networkWhileAbsent"] = self.wait_network(False)
+            self.result["restoredNetwork"] = self.restore_network()
+        before_start = self.http("/__m09")
+        initial_gets = self.http("/__control")["getRequests"]
+        assert (before_start["mutationRequests"], before_start["applied"], before_start["duplicates"]) == ((0, 0, 0) if case["name"] == "A" else (1, 1, 0))
+        self.result["coldLaunchCommand"] = self.command_count + 1
+        self.result["coldLaunchStartedMonotonic"] = time.monotonic()
+        # No install, clear, seed, instrumentation, native write, or manual Sync after death.
+        self.adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        self.completed_ui([case["title"]])
+        self.wait_text("Fresh local data" if self.fixed else "Stale local data")
+        self.wait_text(f"Pending changes: {0 if self.fixed else 1}")
+        new_pid = self.adb("shell", "pidof", PACKAGE)
+        assert new_pid.isdigit() and new_pid != old_pid
+        self.result["afterPid"] = int(new_pid)
+        final = self.capture("after-startup")
+        remote = self.http("/__m09")
+        controls = self.http("/__control")
+        if self.fixed:
+            assert final["items"] == [self.canonical_item()] and final["pending"] == final["tombstones"] == []
+            assert final["allocator"] == [["pending_mutations", 1]]
+            assert len(final["refreshMetadata"]) == 1 and final["refreshMetadata"][0][0] == 1
+            assert type(final["refreshMetadata"][0][1]) is int and final["refreshMetadata"][0][1] > 0
+            assert len(final["acknowledged"]) == 1
+            receipt = final["acknowledged"][0]
+            assert (receipt["clientMutationId"], receipt["payloadHash"], receipt["statusCode"]) == (case["clientMutationId"], case["payloadHash"], case["statusCode"])
+            assert json.loads(receipt["responseBody"]) == {"item": self.canonical_item()}
+            assert remote["items"] == [self.canonical_item()]
+            assert (remote["applied"], remote["duplicates"], remote["mutationRequests"]) == ((1, 0, 1) if case["name"] == "A" else (1, 1, 2))
+        else:
+            assert final == stopped, "Unchanged M08 unexpectedly changed durable state at startup"
+            assert remote["items"] == before_start["items"]
+            for key in ("applied", "duplicates", "mutationRequests", "results", "droppedResponses"):
+                assert remote[key] == before_start[key], (key, remote)
+            assert controls["getRequests"] == initial_gets, "Unchanged M08 unexpectedly made startup HTTP"
+        assert remote["versionConflicts"] == remote["identityConflicts"] == remote["hashRejections"] == 0
+        for entry in remote["requests"]:
+            assert (entry["method"], entry["path"], entry["status"]) == (case["method"], case["path"], case["statusCode"])
+            assert entry["request"] == self.wire() and entry["canonical"] == self.canonical_request()
+            assert entry["actualPayloadHash"] == case["payloadHash"]
+        assert len(remote["requests"]) == remote["mutationRequests"]
+        fixture_bytes = self.args.fixture_log.read_bytes()
+        events = [json.loads(line) for line in fixture_bytes[self.fixture_offset:].splitlines() if line.startswith(b"{")]
+        mutation_events = [event for event in events if "delivery" in event and event["method"] in ("POST", "PATCH", "DELETE") and event["path"].startswith("/items")]
+        assert len(mutation_events) == remote["mutationRequests"]
+        assert mutation_events == remote["requests"]
+        self.result.update(status="PASS", observed="startup recovery" if self.fixed else "M08 startup leaves durable work pending",
+                           final=final, remote=remote, fixtureControls=controls,
+                           fixtureLogEndOffset=len(fixture_bytes), mutationEvents=mutation_events,
+                           elapsedSeconds=round(time.monotonic() - started, 3))
+
+    def cleanup(self):
+        try:
+            super().cleanup()
+        finally:
+            state = self.http("/__m09")
+            barrier = state["barrier"]
+            if barrier is not None and barrier["outcome"] == "held" and not barrier["releaseRequested"]:
+                self.http("/__m09/release", {"action": "drop"})
+                self.result["cleanupDroppedHeldResponse"] = True
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument("--expect", choices=("limitation", "recovered"), required=True)
+    parser.add_argument("--apk", type=Path, required=True)
+    parser.add_argument("--test-apk", type=Path, required=True)
+    parser.add_argument("--output", type=Path, required=True)
+    parser.add_argument("--fixture-log", type=Path, required=True)
+    parser.add_argument("--schema-version", type=int, choices=(5,), default=5)
+    parser.add_argument("--adb", type=Path, default=Path(os.environ.get("ANDROID_HOME", "")) / "platform-tools/adb")
+    args = parser.parse_args()
+    output = args.output.resolve()
+    output.mkdir(parents=True, exist_ok=False)
+    results = []
+    try:
+        for index, case in enumerate(INPUTS["cases"]):
+            scenario = StartupRecoveryScenario(argparse.Namespace(**{**vars(args), "output": output / case["name"]}), case, index == 0)
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
+        summary = dict(scenario="M09", expectation=args.expect,
+                       status="PASS" if len(results) == 2 and all(case["status"] == "PASS" for case in results) else "FAIL",
+                       cases=[{key: case.get(key) for key in ("case", "status", "adbCommands", "beforePid", "afterPid", "error", "cleanup")} for case in results],
+                       adbCommands=sum(case["adbCommands"] for case in results))
+        (output / "result.json").write_text(json.dumps(summary, indent=2) + "\n")
+        print(json.dumps(summary, indent=2), flush=True)
+
+
+if __name__ == "__main__":
+    main()


