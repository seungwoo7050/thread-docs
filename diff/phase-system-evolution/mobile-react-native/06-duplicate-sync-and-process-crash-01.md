# M06 — 중복 Sync와 Process Crash

## `test(M06): reproduce lost acknowledgement on verified M05`

diff --git a/SPEC_REVISION b/SPEC_REVISION
index 81ee29e..c839290 100644
--- a/SPEC_REVISION
+++ b/SPEC_REVISION
@@ -1 +1 @@
-ed7baa246ee947c6778e0f84751c9b91cec7abfe
+61280dd86ce88b6e431f408241c0998a275960aa
diff --git a/fixture/server.cjs b/fixture/server.cjs
index 6f95e6a..f6b8b8e 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,5 +1,15 @@
-// A disposable deterministic M03/M04/M05 fixture, not a backend service.
+// A disposable deterministic M03–M06 fixture, not a backend service.
 const http = require('node:http');
+const {createHash} = require('node:crypto');
+
+// Independent server implementation of the shared compact JSON hash contract.
+function canonicalJson(value) {
+  if (Array.isArray(value)) {return `[${value.map(canonicalJson).join(',')}]`;}
+  if (value !== null && typeof value === 'object') {
+    return `{${Object.keys(value).sort().map(key => `${JSON.stringify(key)}:${canonicalJson(value[key])}`).join(',')}}`;
+  }
+  return JSON.stringify(value);
+}
 
 const SEEDS = [
   {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
@@ -11,12 +21,18 @@ function createFixture() {
   let nextTimestamp;
   let requests;
   let getFailures;
+  let receipts;
+  let mutationEvidence;
+  let dropFirstCreate;
   const snapshot = () => [...items.values()].sort((a, b) => a.id.localeCompare(b.id));
   function reset() {
     items = new Map(SEEDS.map(item => [item.id, {...item}]));
     nextTimestamp = 1700000100000;
     requests = [];
     getFailures = 0;
+    receipts = new Map();
+    mutationEvidence = {appliedCount: 0, duplicateCount: 0, conflictCount: 0, hashRejectedCount: 0, attempts: []};
+    dropFirstCreate = false;
   }
   reset();
   const state = () => ({items: snapshot(), nextTimestamp, requests});
@@ -24,9 +40,29 @@ function createFixture() {
 
   const server = http.createServer(async (request, response) => {
     let body;
+    let wireBody;
+    let mutation;
+    let clientMutationId;
     const {method, url: path} = request;
-    function reply(status, payload) {
+    function reply(status, payload, outcome = 'rejected') {
       if (!path.startsWith('/__')) {requests.push({method, path, body: body ?? null, status, response: payload});}
+      const applied = mutation && status >= 200 && status < 300 && outcome !== 'duplicate';
+      if (applied) {
+        mutationEvidence.appliedCount += 1;
+        outcome = 'applied';
+        if (clientMutationId) {receipts.set(clientMutationId, {canonical: mutation.canonical, status, response: payload});}
+      }
+      const responseDropped = Boolean(applied && dropFirstCreate && method === 'POST' && path === '/items' && body.id === 'crash-001');
+      if (mutation) {mutationEvidence.attempts.push({...mutation, outcome, status, response: payload, responseDropped});}
+      if (responseDropped) {
+        dropFirstCreate = false;
+        // Flush real response headers, then close after one byte. The declared
+        // full length cannot be read, so the client has no acknowledged JSON.
+        const encoded = Buffer.from(JSON.stringify(payload));
+        response.writeHead(status, {'Content-Type': 'application/json', 'Content-Length': encoded.length, Connection: 'close'});
+        response.flushHeaders();
+        return response.end(encoded.subarray(0, 1));
+      }
       response.writeHead(status, {'Content-Type': 'application/json'});
       response.end(JSON.stringify(payload));
     }
@@ -35,8 +71,17 @@ function createFixture() {
       for await (const chunk of request) {chunks.push(chunk);}
       const raw = Buffer.concat(chunks).toString('utf8');
       if (raw) {body = JSON.parse(raw);}
+      wireBody = body ?? null;
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
+      if (method === 'GET' && path === '/__mutation-state') {return reply(200, mutationEvidence);}
+      if (method === 'POST' && path === '/__m06-reset') {
+        reset();
+        items.clear();
+        nextTimestamp = 1700000400000;
+        dropFirstCreate = true;
+        return reply(200, {...state(), dropFirstCreate, delayMs: 0});
+      }
       if (method === 'POST' && path === '/__mutation-clock') {
         if (!body || !Number.isSafeInteger(body.nextTimestamp) || body.nextTimestamp < 0) {
           return reply(400, {error: 'Nonnegative mutation timestamp is required'});
@@ -67,6 +112,42 @@ function createFixture() {
         if (getFailures > 0) {getFailures -= 1; return reply(503, {error: 'Temporary GET failure'});}
         return reply(200, {items: snapshot()});
       }
+      if ((method === 'POST' && path === '/items') || ((method === 'PATCH' || method === 'DELETE') && /^\/items\/[^/]+$/.test(path))) {
+        const hasIdentity = body && Object.hasOwn(body, 'clientMutationId');
+        const hasHash = body && Object.hasOwn(body, 'payloadHash');
+        clientMutationId = body?.clientMutationId;
+        const declaredHash = body?.payloadHash;
+        if (hasIdentity || hasHash) {
+          body = {...body};
+          delete body.clientMutationId;
+          delete body.payloadHash;
+          if (method === 'DELETE' && Object.keys(body).length === 0) {body = null;}
+        }
+        const canonical = canonicalJson({method, path, payload: body ?? null});
+        const actualHash = createHash('sha256').update(canonical, 'utf8').digest('hex');
+        mutation = {method, path, wireBody, clientMutationId: clientMutationId ?? null,
+          declaredHash: declaredHash ?? null, actualHash, canonical};
+        // Legacy requests remain available for unchanged M05 reproduction and
+        // earlier fixed scenarios. M06 clients must always provide both fields.
+        if (hasIdentity || hasHash) {
+          if (typeof clientMutationId !== 'string' || !clientMutationId || typeof declaredHash !== 'string') {
+            return reply(400, {error: 'mutation_identity_required'});
+          }
+          if (declaredHash !== actualHash) {
+            mutationEvidence.hashRejectedCount += 1;
+            return reply(400, {error: 'payload_hash_mismatch'}, 'hash_mismatch');
+          }
+          const previous = receipts.get(clientMutationId);
+          if (previous) {
+            if (previous.canonical !== canonical) {
+              mutationEvidence.conflictCount += 1;
+              return reply(409, {error: 'identity_conflict'}, 'identity_conflict');
+            }
+            mutationEvidence.duplicateCount += 1;
+            return reply(previous.status, previous.response, 'duplicate');
+          }
+        }
+      }
       if (method === 'POST' && path === '/items') {
         if (!body || typeof body.id !== 'string' || !/^[a-zA-Z0-9_-]+$/.test(body.id)
             || typeof body.title !== 'string' || !body.title.trim() || typeof body.completed !== 'boolean') {
@@ -102,7 +183,7 @@ function createFixture() {
       return reply(400, {error: 'Invalid JSON request'});
     }
   });
-  return {server, reset, state};
+  return {server, reset, state, mutationState: () => mutationEvidence};
 }
 
 module.exports = {createFixture};
diff --git a/scripts/verify_m06.py b/scripts/verify_m06.py
new file mode 100644
index 0000000..30fbea9
--- /dev/null
+++ b/scripts/verify_m06.py
@@ -0,0 +1,363 @@
+#!/usr/bin/env python3
+"""Frozen M06 lost acknowledgement, actual process death, replay and collision.
+
+Exclusive device lease required. Baseline is the unchanged M05 APK; only its
+pre-scenario native schema3 seed is external. The fixed APK creates via the UI.
+"""
+import argparse
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import sqlite3
+import subprocess
+import time
+from urllib.error import HTTPError
+from urllib.request import Request, urlopen
+import xml.etree.ElementTree as ET
+
+PACKAGE = "com.mse.reactnative"
+URL = "http://127.0.0.1:18081"
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--node", default="node")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    parser.add_argument("--baseline", action="store_true")
+    args = parser.parse_args()
+    project = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs_path = project / "verification/M06-inputs.json"
+    inputs = json.loads(inputs_path.read_text())
+    canonical = inputs["canonicalItem"]
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "host_pid": os.getpid(), "serial": args.serial,
+              "apk_sha256": hashlib.sha256(Path(args.apk).read_bytes()).hexdigest(),
+              "harness_sha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+              "fixture_sha256": hashlib.sha256((project / "fixture/server.cjs").read_bytes()).hexdigest(),
+              "inputs_sha256": hashlib.sha256(inputs_path.read_bytes()).hexdigest()}
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(*parts, check=True, binary=False):
+        command = [args.adb, "-s", args.serial, *parts]
+        completed = subprocess.run(command, capture_output=True, timeout=60)
+        commands.append({"command": command, "exit": completed.returncode,
+                         "stdout": "<binary>" if binary else completed.stdout.decode(errors="replace"),
+                         "stderr": completed.stderr.decode(errors="replace")})
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        if check and completed.returncode:
+            raise AssertionError(commands[-1])
+        return completed.stdout if binary else completed.stdout.decode(errors="replace").strip()
+
+    def remote(path="/__state", body=None, expected=200):
+        method = "GET" if body is None else "POST"
+        event = {"method": method, "path": path, "body": body}
+        try:
+            request = Request(URL + path, data=None if body is None else json.dumps(body).encode(),
+                              headers={"Content-Type": "application/json"}, method=method)
+            try:
+                response = urlopen(request, timeout=3)
+            except HTTPError as error:
+                response = error
+            with response:
+                value = json.load(response)
+                event.update(status=response.status, response=value)
+            assert event["status"] == expected, event
+            return value
+        except Exception as error:
+            event["error"] = repr(error)
+            raise
+        finally:
+            controls.append(event)
+            (evidence / "http-controls.json").write_text(json.dumps(controls, indent=2) + "\n")
+
+    def snapshot(name=None):
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m06-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m06-ui.xml")
+        if name:
+            (evidence / f"{name}.xml").write_text(xml)
+            (evidence / f"{name}.png").write_bytes(adb("exec-out", "screencap", "-p", binary=True))
+        return ET.fromstring(xml)
+
+    def find(label, attribute="content-desc"):
+        deadline = time.monotonic() + 20
+        while time.monotonic() < deadline:
+            for node in snapshot().iter("node"):
+                if node.get(attribute) == label:
+                    return node
+        raise AssertionError(f"Missing {attribute}: {label}")
+
+    def tap(label):
+        node = find(label)
+        assert node.get("enabled") != "false", label
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", node.get("bounds")))
+        adb("shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
+
+    def create(title):
+        tap("New item title")
+        adb("shell", "input", "text", title.replace(" ", "%s"))
+        adb("shell", "input", "keyevent", "KEYCODE_BACK")
+        tap("Add item")
+        find("Local storage ready")
+        find("Item count: 1")
+        find(title, "text")
+
+    def launch():
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--ez", "m06FixedIdentity", "true")
+        find("Local storage ready")
+
+    def database(name):
+        files = adb("shell", "run-as", PACKAGE, "ls", "files").splitlines()
+        path = evidence / f"{name}.db"
+        for suffix in ("", "-wal", "-shm"):
+            if "items.db" + suffix in files:
+                Path(str(path) + suffix).write_bytes(adb("exec-out", "run-as", PACKAGE, "cat", f"files/items.db{suffix}", binary=True))
+        assert path.exists(), "Native SQLite file missing"
+        with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
+            assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+            schema = db.execute("PRAGMA user_version").fetchone()[0]
+            assert schema == (3 if args.baseline else 4), schema
+            items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
+                     for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
+            columns = "sequence, kind, item_id, payload" + ("" if args.baseline else ", client_mutation_id, payload_hash, terminal_error")
+            pending = []
+            for row in db.execute(f"SELECT {columns} FROM pending_mutations ORDER BY sequence"):
+                intent = {"sequence": row[0], "kind": row[1], "itemId": row[2], "payload": None if row[3] is None else json.loads(row[3])}
+                if not args.baseline:
+                    intent.update(clientMutationId=row[4], payloadHash=row[5], terminalError=row[6])
+                pending.append(intent)
+            ack = None if args.baseline else db.execute("SELECT last_acknowledgement FROM sync_metadata WHERE singleton=1").fetchone()[0]
+            value = {"schema_version": schema, "items": items, "pending": pending,
+                     "last_acknowledgement": None if ack is None else json.loads(ack),
+                     "last_successful_refresh_at": db.execute("SELECT last_successful_refresh_at FROM sync_metadata WHERE singleton=1").fetchone()[0],
+                     "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+        (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
+        return value
+
+    def expected_intent(collision=False):
+        payload = inputs["collisionPayload" if collision else "payload"]
+        intent = {"sequence": 1, "kind": "create", "itemId": "crash-001", "payload": payload}
+        if not args.baseline:
+            intent.update(clientMutationId=inputs["clientMutationId"],
+                          payloadHash=inputs["collisionHash" if collision else "payloadHash"], terminalError=None)
+        return intent
+
+    def network_state(name):
+        state = {key: adb("shell", "settings", "get", "global", key)
+                 for key in ("airplane_mode_on", "wifi_on", "mobile_data")}
+        connectivity = adb("shell", "dumpsys", "connectivity")
+        (evidence / f"{name}-connectivity.txt").write_text(connectivity)
+        assert "Active default network: none" not in connectivity
+        return state
+
+    def baseline_seed():
+        # Approved setup strictly BEFORE the tested lost-ack/death boundary.
+        adb("shell", "am", "force-stop", PACKAGE)
+        assert not adb("shell", "pidof", PACKAGE, check=False)
+        assert database("baseline-empty")["items"] == []
+        seed = evidence / "baseline-seed.db"
+        with sqlite3.connect(evidence / "baseline-empty.db") as source, sqlite3.connect(seed) as db:
+            source.backup(db)
+            db.execute("PRAGMA journal_mode=DELETE")
+            db.execute("INSERT INTO items VALUES (?, ?, ?, ?, ?)",
+                       ("crash-001", "Crash safe", 0, 1, inputs["baselineLocalTimestamp"]))
+            db.execute("INSERT INTO pending_mutations (kind, item_id, payload) VALUES (?, ?, ?)",
+                       ("create", "crash-001", json.dumps(inputs["payload"], separators=(",", ":"))))
+            db.execute("UPDATE local_identity SET next_id=2 WHERE singleton=1")
+            db.commit()
+        seed.chmod(0o644)
+        result["baseline_seed_sha256"] = hashlib.sha256(seed.read_bytes()).hexdigest()
+        adb("push", str(seed), "/data/local/tmp/mse-m06-seed.db")
+        adb("shell", "run-as", PACKAGE, "rm", "-f", "files/items.db-wal", "files/items.db-shm")
+        adb("shell", "run-as", PACKAGE, "cp", "/data/local/tmp/mse-m06-seed.db", "files/items.db")
+        adb("shell", "rm", "/data/local/tmp/mse-m06-seed.db")
+        result["baseline_seed_last_command_index"] = len(commands)
+        launch()
+
+    fixture = None
+    original_network = None
+    fixture_log = (evidence / "fixture.log").open("w")
+    try:
+        fixture_command = [args.node, "fixture/server.cjs"]
+        result["fixture_command"] = fixture_command
+        fixture = subprocess.Popen(fixture_command, cwd=project, stdout=fixture_log, stderr=subprocess.STDOUT)
+        result["fixture_pid"] = fixture.pid
+        deadline = time.monotonic() + 10
+        while True:
+            assert fixture.poll() is None, "Owned fixture could not start"
+            try:
+                remote()
+                break
+            except Exception:
+                assert time.monotonic() < deadline, "Fixture did not become ready"
+                time.sleep(0.05)
+        reset = remote("/__m06-reset", {})
+        assert reset == {"items": [], "nextTimestamp": 1700000400000, "requests": [], "dropFirstCreate": True, "delayMs": 0}
+        original_network = network_state("initial")
+        result["network_before"] = original_network
+        assert original_network == {"airplane_mode_on": "0", "wifi_on": "1", "mobile_data": "1"}
+        adb("install", "-r", args.apk)
+        adb("shell", "am", "force-stop", PACKAGE)
+        assert adb("shell", "pm", "clear", PACKAGE) == "Success"
+        adb("shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("shell", "wm", "dismiss-keyguard")
+        launch()
+        assert database("initial-empty")["items"] == []
+        if args.baseline:
+            baseline_seed()
+        else:
+            create("Crash safe")
+        find("Pending uploads: 1")
+        result["before_send"] = database("before-send")
+        assert result["before_send"]["pending"] == [expected_intent()]
+        assert result["before_send"]["last_acknowledgement"] is None
+        assert [{key: item[key] for key in ("id", "title", "completed", "version")} for item in result["before_send"]["items"]] == [
+            {"id": "crash-001", "title": "Crash safe", "completed": False, "version": 1}]
+        assert remote()["requests"] == []
+        snapshot("before-send")
+        tap("Synchronize")
+        find("Sync status: error")
+        find("Local storage ready")
+        find("Pending uploads: 1")
+        result["after_lost_ack"] = database("after-lost-ack")
+        assert result["after_lost_ack"] == result["before_send"]
+        snapshot("lost-ack-error")
+        first = remote("/__mutation-state")
+        result["first_attempt"] = first
+        assert first["appliedCount"] == 1 and first["duplicateCount"] == 0 and len(first["attempts"]) == 1
+        attempt = first["attempts"][0]
+        assert attempt["responseDropped"] is True and attempt["status"] == 201
+        assert attempt["response"] == {"item": canonical} and attempt["actualHash"] == inputs["payloadHash"]
+        assert attempt["wireBody"] == (inputs["payload"] if args.baseline else inputs["wireBody"])
+
+        result["death_boundary_start_command_index"] = len(commands)
+        result["pid_before"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_before"]
+        adb("shell", "am", "force-stop", PACKAGE)
+        result["pid_after_kill"] = adb("shell", "pidof", PACKAGE, check=False)
+        assert not result["pid_after_kill"]
+        assert database("after-kill") == result["after_lost_ack"]
+        launch()
+        result["pid_after_restart"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_after_restart"] and result["pid_after_restart"] != result["pid_before"]
+        result["after_restart"] = database("after-restart")
+        assert result["after_restart"] == result["after_lost_ack"]
+        find("Pending uploads: 1")
+        assert remote("/__mutation-state") == first, "Restart must not automatically drain"
+        result["death_boundary_end_command_index"] = len(commands)
+        assert os.getpid() == result["host_pid"]
+        save()
+
+        tap("Synchronize")
+        find("Sync status: error" if args.baseline else "Sync status: fresh")
+        find("Local storage ready")
+        find("Pending uploads: 1" if args.baseline else "Pending uploads: 0")
+        result["after_replay"] = database("after-replay")
+        result["remote_after_replay"] = remote()
+        replay = remote("/__mutation-state")
+        result["replay_evidence"] = replay
+        assert result["remote_after_replay"]["items"] == [canonical]
+        assert result["remote_after_replay"]["nextTimestamp"] == 1700000401000
+        assert replay["appliedCount"] == 1 and len(replay["attempts"]) == 2
+        a, b = replay["attempts"]
+        assert a["wireBody"] == b["wireBody"] and a["actualHash"] == b["actualHash"] == inputs["payloadHash"]
+        assert b["responseDropped"] is False
+        if args.baseline:
+            assert b["status"] == 409 and b["response"] == {"error": "Item already exists"}
+            assert replay["duplicateCount"] == 0
+            assert result["after_replay"] == result["after_lost_ack"]
+            assert all(attempt["clientMutationId"] is None and attempt["declaredHash"] is None for attempt in replay["attempts"])
+            result["status"] = "BASELINE_LIMIT_REPRODUCED"
+            result["limitation"] = "M05 applied the first legacy create but lost its response; replay after real process death returns409, retains pending1 and cannot reconcile. Neither request has durable mutation identity/hash."
+        else:
+            assert replay["duplicateCount"] == 1
+            assert b["outcome"] == "duplicate" and b["status"] == a["status"] == 201 and b["response"] == a["response"]
+            assert result["after_replay"]["items"] == [canonical] and result["after_replay"]["pending"] == []
+            assert result["after_replay"]["last_acknowledgement"] == inputs["acknowledgement"]
+            result["primary_status"] = "PASS"
+        snapshot("after-replay")
+        save()
+
+        if not args.baseline:
+            # Isolated collision phase, AFTER the primary recovery has passed.
+            # Fresh local DB creates the alternate body through production code.
+            result["collision_setup_command_index"] = len(commands)
+            adb("shell", "am", "force-stop", PACKAGE)
+            assert adb("shell", "pm", "clear", PACKAGE) == "Success"
+            launch()
+            create("Different payload")
+            assert database("collision-before-send")["pending"] == [expected_intent(True)]
+            tap("Synchronize")
+            find("Sync status: error")
+            find("Local storage ready")
+            blocked = database("collision-terminal")
+            assert blocked["pending"] == [{**expected_intent(True), "terminalError": "identity_conflict"}]
+            assert blocked["last_acknowledgement"] is None
+            collision = remote("/__mutation-state")
+            assert collision["appliedCount"] == 1 and collision["duplicateCount"] == 1 and collision["conflictCount"] == 1
+            assert len(collision["attempts"]) == 3
+            assert collision["attempts"][-1]["wireBody"] == inputs["collisionWireBody"]
+            assert collision["attempts"][-1]["status"] == 409 and collision["attempts"][-1]["response"] == {"error": "identity_conflict"}
+            result["collision_pid_before"] = adb("shell", "pidof", PACKAGE)
+            adb("shell", "am", "force-stop", PACKAGE)
+            assert not adb("shell", "pidof", PACKAGE, check=False)
+            launch()
+            result["collision_pid_after"] = adb("shell", "pidof", PACKAGE)
+            assert result["collision_pid_after"] != result["collision_pid_before"]
+            assert database("collision-reopened") == blocked
+            tap("Synchronize")
+            find("Sync status: error")
+            find("Local storage ready")
+            assert database("collision-no-retry") == blocked
+            assert remote("/__mutation-state") == collision, "Terminal identity conflict was retried"
+            assert remote("/items", inputs["wrongHashWireBody"], expected=400) == {"error": "payload_hash_mismatch"}
+            result["isolated_collision_evidence"] = remote("/__mutation-state")
+            assert result["isolated_collision_evidence"]["hashRejectedCount"] == 1
+            assert result["isolated_collision_evidence"]["appliedCount"] == 1
+            assert result["isolated_collision_evidence"]["duplicateCount"] == 1
+            assert len(result["isolated_collision_evidence"]["attempts"]) == 4
+            assert remote()["items"] == [canonical] and remote()["nextTimestamp"] == 1700000401000
+            snapshot("collision-not-retried")
+            result["status"] = "PASS"
+    except Exception as error:
+        result["status"], result["error"] = "FAIL", repr(error)
+    finally:
+        if original_network is not None:
+            try:
+                # M06 never changes connectivity; verify the recorded initial state.
+                result["network_after"] = network_state("cleanup")
+                assert result["network_after"] == original_network
+                adb("shell", "am", "force-stop", PACKAGE)
+                result["pid_after_cleanup"] = adb("shell", "pidof", PACKAGE, check=False)
+                assert not result["pid_after_cleanup"]
+            except Exception as error:
+                result["status"], result["cleanup_error"] = "FAIL", repr(error)
+        if fixture is not None:
+            if fixture.poll() is None:
+                fixture.terminate()
+            try:
+                result["fixture_exit"] = fixture.wait(timeout=10)
+            except subprocess.TimeoutExpired:
+                fixture.kill()
+                result["fixture_exit"] = fixture.wait(timeout=10)
+                result["status"], result["fixture_cleanup_error"] = "FAIL", "Owned fixture needed forced cleanup"
+            if result["fixture_exit"] != 0:
+                result["status"], result["fixture_cleanup_error"] = "FAIL", "Owned fixture did not exit cleanly"
+        fixture_log.close()
+        result["adb_command_count"] = len(commands)
+        save()
+        print(json.dumps(result, indent=2), flush=True)
+    if result["status"] == "FAIL":
+        raise AssertionError(result.get("error", result.get("cleanup_error", "M06 did not complete")))
+
+
+if __name__ == "__main__":
+    main()
diff --git a/verification/M06-inputs.json b/verification/M06-inputs.json
new file mode 100644
index 0000000..e3c5c03
--- /dev/null
+++ b/verification/M06-inputs.json
@@ -0,0 +1,143 @@
+{
+  "thread": "M06",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "95651acd58066f7b9c02d511073149328172136f",
+  "scenarioSha256": "d636b1a8862100db8b35d8ceccba1573dbbcbead44f0d8bb745d79dbd45b7af1",
+  "clientMutationId": "m06-create-001",
+  "payload": {
+    "id": "crash-001",
+    "title": "Crash safe",
+    "completed": false
+  },
+  "canonicalJson": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Crash safe\"}}",
+  "payloadHash": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba",
+  "wireBody": {
+    "id": "crash-001",
+    "title": "Crash safe",
+    "completed": false,
+    "clientMutationId": "m06-create-001",
+    "payloadHash": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba"
+  },
+  "collisionPayload": {
+    "id": "crash-001",
+    "title": "Different payload",
+    "completed": false
+  },
+  "collisionCanonicalJson": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Different payload\"}}",
+  "collisionHash": "3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73",
+  "collisionWireBody": {
+    "id": "crash-001",
+    "title": "Different payload",
+    "completed": false,
+    "clientMutationId": "m06-create-001",
+    "payloadHash": "3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73"
+  },
+  "wrongHashWireBody": {
+    "id": "crash-001",
+    "title": "Different payload",
+    "completed": false,
+    "clientMutationId": "m06-create-001",
+    "payloadHash": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba"
+  },
+  "canonicalItem": {
+    "id": "crash-001",
+    "title": "Crash safe",
+    "completed": false,
+    "version": 1,
+    "updatedAt": 1700000400000
+  },
+  "acknowledgement": {
+    "clientMutationId": "m06-create-001",
+    "payloadHash": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba",
+    "status": 201,
+    "result": {
+      "item": {
+        "id": "crash-001",
+        "title": "Crash safe",
+        "completed": false,
+        "version": 1,
+        "updatedAt": 1700000400000
+      }
+    }
+  },
+  "baselineLocalTimestamp": 1700000399000,
+  "nextRemoteTimestamp": 1700000400000,
+  "delayMs": 0,
+  "dropPoint": "Commit first crash-001 create, flush status201 headers/full Content-Length, end response after first JSON byte.",
+  "baselineSetup": "Unchanged M05 native schema3 empty database; pre-boundary seed exact crash-001 and its legacy pending create, allocator2; no identity/hash columns. No native state rewrite across tested death boundary.",
+  "fixedSetup": "Debug launch m06FixedIdentity controls only Item prefix crash and mutation identity m06-create-001. Production UI create and native transaction persist the Item and intent.",
+  "isolatedCollision": "After primary result is saved, clear local app data only; production UI creates crash-001/Different payload with the same controlled identity; persist terminal409, reopen and one explicit drain must not resend. Then POST altered body with original wrong hash.",
+  "hashVectors": [
+    {
+      "method": "POST",
+      "path": "/items",
+      "payload": {
+        "id": "crash-001",
+        "title": "Crash safe",
+        "completed": false
+      },
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Crash safe\"}}",
+      "sha256": "09d1c3d2db9864ea761ce798ad13b1ee10faba85a8500b552467408d8245d2ba"
+    },
+    {
+      "method": "POST",
+      "path": "/items",
+      "payload": {
+        "id": "crash-001",
+        "title": "Different payload",
+        "completed": false
+      },
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"crash-001\",\"title\":\"Different payload\"}}",
+      "sha256": "3578ce895fdfea0354ac912266ee46823b075efb9a4f54f07333fe6f101bda73"
+    },
+    {
+      "method": "DELETE",
+      "path": "/items/crash-001",
+      "payload": null,
+      "canonical": "{\"method\":\"DELETE\",\"path\":\"/items/crash-001\",\"payload\":null}",
+      "sha256": "443d17b7938e7859a5b26a7d68fc6b5d4a6f9f1006294bdb97439de3f5610535"
+    },
+    {
+      "method": "PATCH",
+      "path": "/items/remote-001",
+      "payload": {
+        "title": "한글 café ☃ 🧪 \"quote\" \\ slash /\n\t\u0001",
+        "completed": true
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/remote-001\",\"payload\":{\"completed\":true,\"title\":\"한글 café ☃ 🧪 \\\"quote\\\" \\\\ slash /\\n\\t\\u0001\"}}",
+      "sha256": "39a9a53e5bd526b217b8a59f64308b7764d01cd4889981d7e764a978f29ce849"
+    },
+    {
+      "method": "POST",
+      "path": "/items",
+      "payload": {
+        "z": [
+          {
+            "z": 2,
+            "a": 1
+          },
+          null,
+          false,
+          [
+            "x",
+            {
+              "b": 0,
+              "a": -1
+            }
+          ]
+        ],
+        "a": {
+          "10": 10,
+          "2": 2,
+          "nested": {
+            "b": false,
+            "a": true
+          }
+        }
+      },
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"a\":{\"10\":10,\"2\":2,\"nested\":{\"a\":true,\"b\":false}},\"z\":[{\"a\":1,\"z\":2},null,false,[\"x\",{\"a\":-1,\"b\":0}]]}}",
+      "sha256": "80de31aca00d40683eb4291cf9c8f10239fa84c15ba0ed98f006e92b51dced88"
+    }
+  ]
+}
diff --git a/verification/M06.md b/verification/M06.md
new file mode 100644
index 0000000..8fd23b5
--- /dev/null
+++ b/verification/M06.md
@@ -0,0 +1,73 @@
+# M06 verification — phase-1, attempt 1
+
+- Branch `track/react-native`; START `95651acd58066f7b9c02d511073149328172136f`.
+- SPEC_REVISION `61280dd86ce88b6e431f408241c0998a275960aa`.
+- Common scenario SHA-256 `d636b1a8862100db8b35d8ceccba1573dbbcbead44f0d8bb745d79dbd45b7af1`.
+- Raw root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/`.
+- Android: the shared API34 Pixel6 ARM64 emulator `emulator-5554`.
+
+## Inputs and unchanged M05 reproduction
+
+HEAD, branch and peeled `progress/phase-1/react-native/M05` matched START. The
+profile transition preserved the paused supporting fixture/harness and prior
+evidence; no application edit or old-revision M06 commit occurred. See
+[profile-start.json](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/profile-start.json).
+The original baseline source/APK and discovery failure remain at the referenced
+older evidence root; they were not overwritten or needlessly rebuilt.
+
+The verified M05 APK SHA-256 is
+`6698265d6cd2e7d4517b404a8287b41f3d557d5dc346b100cf24e1dde96012a2`.
+Before the first baseline execution, all44 other tracked files were unchanged,
+and these supporting inputs were frozen in
+[frozen-inputs-before-baseline.json](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/frozen-inputs-before-baseline.json):
+
+| Input | SHA-256 |
+|---|---|
+| `fixture/server.cjs` | `7915b355cd8be0b014cdc50b71275e009b25d2be3af5b5c1c58c80e90cdbe38c` |
+| `scripts/verify_m06.py` | `17263654d2e377da162f227f09c00a8d516f4ad7f7998cb57dc67a615610a4ff` |
+| `verification/M06-inputs.json` | `094573f61087722760246e2fb98aa7905547d51c5e5f7e4c993532add7258a59` |
+
+The fixture independently hashes actual request payloads, retains original
+results by identity, and records raw wire bodies separately from prior domain
+traces. Legacy requests remain supported for unchanged baseline/regressions.
+The first crash-001 create commits, flushes201 headers with full Content-Length,
+then closes after one JSON byte. No usable acknowledgment reaches the client.
+Five independent Python/hashlib vectors were frozen before testing; main also
+recomputed them. No fixture delay, failure point or success condition was tuned.
+
+Under main's exclusive baseline-only lease, from the branch root:
+
+```sh
+python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/run.py baseline-android-01 python3 scripts/verify_m06.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M06/m05-baseline.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/baseline-android-01 --baseline
+```
+
+**BASELINE_LIMIT_REPRODUCED**, exit0,48.605seconds,79 adb commands. Main approved
+the schema3 pre-test seed of crash-001/Crash safe/false and its exact legacy
+pending-create row because the unchanged APK cannot select the fixed crash ID.
+All seed/install/clear operations finished before the tested death boundary.
+First POST applied once, lost its response, and retained pending1. App PID7916 →
+absent → PID8230; native Items/intent/allocator/metadata survived identically.
+No state injection occurred across command indexes46–59. The one replay returned
+409 `Item already exists`, leaving pending1, applied1, duplicate0. Both actual
+requests lacked identity/hash; this is the unmodified M05 guarantee's limit.
+
+[Full result, wire bodies and process evidence](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/baseline-android-01/result.json)
+and adjacent `commands.json`, native SQLite copies, XML/PNG and fixture log are
+preserved. App PID was absent at cleanup, owned fixture42353 exited0, and unchanged
+network0/1/1 plus an active default network were verified. The lease was released;
+no implemented-device run is claimed.
+
+## Invocation record
+
+Every named invocation has immutable `.command.json`/`.log` and a source snapshot
+under the raw root. The earlier malformed read failure is retained at
+`evidence/react-native/M06/discovery-01.*`; no tests ran before the profile change.
+
+| Invocation | Result |
+|---|---|
+| `fixture-syntax-01` | `node --check fixture/server.cjs`, exit0 |
+| `harness-syntax-01` | Python AST syntax check, exit0 |
+| `baseline-android-01` | Expected M05 lost-ack limitation and cleanup verified, exit0 |
+
+Implementation, host checks and main's final fixed Android run remain pending at
+this reproduction commit. No M07 or phase-2 feature is included.


