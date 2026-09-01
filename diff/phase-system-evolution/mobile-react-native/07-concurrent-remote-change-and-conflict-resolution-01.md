# M07 — Concurrent Remote Change와 Conflict Resolution

## `test(M07): preserve frozen baseline evidence and setup failure`

diff --git a/fixture/server.cjs b/fixture/server.cjs
index f6b8b8e..6f2d056 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03–M06 fixture, not a backend service.
+// A disposable deterministic M03–M07 fixture, not a backend service.
 const http = require('node:http');
 const {createHash} = require('node:crypto');
 
@@ -24,7 +24,10 @@ function createFixture() {
   let receipts;
   let mutationEvidence;
   let dropFirstCreate;
+  let dropMutationId;
+  let tombstones;
   const snapshot = () => [...items.values()].sort((a, b) => a.id.localeCompare(b.id));
+  const deletedSnapshot = () => [...tombstones.values()].sort((a, b) => a.id.localeCompare(b.id));
   function reset() {
     items = new Map(SEEDS.map(item => [item.id, {...item}]));
     nextTimestamp = 1700000100000;
@@ -33,9 +36,12 @@ function createFixture() {
     receipts = new Map();
     mutationEvidence = {appliedCount: 0, duplicateCount: 0, conflictCount: 0, hashRejectedCount: 0, attempts: []};
     dropFirstCreate = false;
+    dropMutationId = null;
+    tombstones = new Map();
   }
   reset();
-  const state = () => ({items: snapshot(), nextTimestamp, requests});
+  const deletionState = () => tombstones.size ? {tombstones: deletedSnapshot()} : {};
+  const state = () => ({items: snapshot(), ...deletionState(), nextTimestamp, requests});
   const tick = () => {const timestamp = nextTimestamp; nextTimestamp += 1000; return timestamp;};
 
   const server = http.createServer(async (request, response) => {
@@ -52,10 +58,15 @@ function createFixture() {
         outcome = 'applied';
         if (clientMutationId) {receipts.set(clientMutationId, {canonical: mutation.canonical, status, response: payload});}
       }
-      const responseDropped = Boolean(applied && dropFirstCreate && method === 'POST' && path === '/items' && body.id === 'crash-001');
+      if (mutation && clientMutationId && outcome === 'version_conflict') {
+        receipts.set(clientMutationId, {canonical: mutation.canonical, status, response: payload});
+      }
+      const responseDropped = Boolean(applied && ((dropFirstCreate && method === 'POST' && path === '/items' && body.id === 'crash-001')
+        || (dropMutationId !== null && clientMutationId === dropMutationId)));
       if (mutation) {mutationEvidence.attempts.push({...mutation, outcome, status, response: payload, responseDropped});}
       if (responseDropped) {
         dropFirstCreate = false;
+        dropMutationId = null;
         // Flush real response headers, then close after one byte. The declared
         // full length cannot be read, so the client has no acknowledged JSON.
         const encoded = Buffer.from(JSON.stringify(payload));
@@ -82,6 +93,17 @@ function createFixture() {
         dropFirstCreate = true;
         return reply(200, {...state(), dropFirstCreate, delayMs: 0});
       }
+      if (method === 'POST' && path === '/__m07-reset') {
+        reset();
+        items = new Map([['conflict-001', {id: 'conflict-001', title: 'Initial', completed: false, version: 1, updatedAt: 1700000000000}]]);
+        nextTimestamp = 1700000501000;
+        return reply(200, {...state(), delayMs: 0});
+      }
+      if (method === 'POST' && path === '/__drop-ack') {
+        if (!body || typeof body.clientMutationId !== 'string' || !body.clientMutationId) {return reply(400, {error: 'Mutation identity is required'});}
+        dropMutationId = body.clientMutationId;
+        return reply(200, {clientMutationId: dropMutationId, delayMs: 0});
+      }
       if (method === 'POST' && path === '/__mutation-clock') {
         if (!body || !Number.isSafeInteger(body.nextTimestamp) || body.nextTimestamp < 0) {
           return reply(400, {error: 'Nonnegative mutation timestamp is required'});
@@ -108,9 +130,17 @@ function createFixture() {
         items.set(item.id, updated);
         return reply(200, {item: updated});
       }
+      if (method === 'POST' && path === '/__remote-delete') {
+        const item = items.get(body?.id);
+        if (!item || !Number.isSafeInteger(body.updatedAt)) {return reply(400, {error: 'Existing id and fixed updatedAt are required'});}
+        const tombstone = {id: item.id, version: item.version + 1, updatedAt: body.updatedAt, deleted: true};
+        items.delete(item.id);
+        tombstones.set(item.id, tombstone);
+        return reply(200, {tombstone});
+      }
       if (method === 'GET' && path === '/items') {
         if (getFailures > 0) {getFailures -= 1; return reply(503, {error: 'Temporary GET failure'});}
-        return reply(200, {items: snapshot()});
+        return reply(200, {items: snapshot(), ...deletionState()});
       }
       if ((method === 'POST' && path === '/items') || ((method === 'PATCH' || method === 'DELETE') && /^\/items\/[^/]+$/.test(path))) {
         const hasIdentity = body && Object.hasOwn(body, 'clientMutationId');
@@ -153,7 +183,7 @@ function createFixture() {
             || typeof body.title !== 'string' || !body.title.trim() || typeof body.completed !== 'boolean') {
           return reply(400, {error: 'id, title, and completed are required'});
         }
-        if (items.has(body.id)) {return reply(409, {error: 'Item already exists'});}
+        if (items.has(body.id) || tombstones.has(body.id)) {return reply(409, {error: 'Item already exists'});}
         const item = {id: body.id, title: body.title.trim(), completed: body.completed, version: 1, updatedAt: tick()};
         items.set(item.id, item);
         return reply(201, {item});
@@ -161,19 +191,30 @@ function createFixture() {
       const match = /^\/items\/([a-zA-Z0-9_-]+)$/.exec(path);
       if (match && (method === 'PATCH' || method === 'DELETE')) {
         const item = items.get(match[1]);
+        // The identity replay check above intentionally precedes current-version
+        // validation. Legacy M06 requests retain their unchanged baseline behavior.
+        const hasBase = body && Object.hasOwn(body, 'baseVersion');
+        if (hasBase) {
+          if (!Number.isSafeInteger(body.baseVersion) || body.baseVersion < 0) {return reply(400, {error: 'Invalid baseVersion'});}
+          if (!item || body.baseVersion !== item.version) {
+            return reply(409, {error: 'version_conflict', item: item ?? null, tombstone: tombstones.get(match[1]) ?? null}, 'version_conflict');
+          }
+        }
         if (!item) {return reply(404, {error: 'Item not found'});}
         if (method === 'DELETE') {
           items.delete(item.id);
-          tick();
+          tombstones.set(item.id, {id: item.id, version: item.version + 1, updatedAt: tick(), deleted: true});
           return reply(200, {deletedId: item.id});
         }
-        if (!body || !Object.keys(body).length
-            || Object.keys(body).some(key => key !== 'title' && key !== 'completed')
-            || ('title' in body && (typeof body.title !== 'string' || !body.title.trim()))
-            || ('completed' in body && typeof body.completed !== 'boolean')) {
+        const changes = body && {...body};
+        if (changes) {delete changes.baseVersion;}
+        if (!changes || !Object.keys(changes).length
+            || Object.keys(changes).some(key => key !== 'title' && key !== 'completed')
+            || ('title' in changes && (typeof changes.title !== 'string' || !changes.title.trim()))
+            || ('completed' in changes && typeof changes.completed !== 'boolean')) {
           return reply(400, {error: 'Changed title or completed is required'});
         }
-        const updated = {...item, ...body, ...(body.title === undefined ? {} : {title: body.title.trim()}),
+        const updated = {...item, ...changes, ...(changes.title === undefined ? {} : {title: changes.title.trim()}),
           version: item.version + 1, updatedAt: tick()};
         items.set(item.id, updated);
         return reply(200, {item: updated});
diff --git a/scripts/verify_m07.py b/scripts/verify_m07.py
new file mode 100644
index 0000000..f7f002a
--- /dev/null
+++ b/scripts/verify_m07.py
@@ -0,0 +1,407 @@
+#!/usr/bin/env python3
+"""Frozen M07 Android stale update/delete, conflict persistence and explicit edit.
+
+An exclusive lease is required. The baseline uses unchanged M06 and substitutes
+only its controlled mutation ID, while stopped and strictly before dispatch.
+"""
+import argparse
+import copy
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import sqlite3
+import subprocess
+import time
+from urllib.request import Request, urlopen
+import xml.etree.ElementTree as ET
+
+PACKAGE = "com.mse.reactnative"
+URL = "http://127.0.0.1:18081"
+OFFLINE = {"airplane_mode_on": "1", "wifi_on": "0", "mobile_data": "0"}
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
+    inputs_path = project / "verification/M07-inputs.json"
+    inputs = json.loads(inputs_path.read_text())
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "host_pid": os.getpid(), "cases": [],
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
+    def remote(path="/__state", body=None):
+        event = {"method": "GET" if body is None else "POST", "path": path, "body": body}
+        try:
+            request = Request(URL + path, data=None if body is None else json.dumps(body).encode(),
+                              headers={"Content-Type": "application/json"}, method=event["method"])
+            with urlopen(request, timeout=3) as response:
+                value = json.load(response)
+                event.update(status=response.status, response=value)
+            assert event["status"] == 200, event
+            return value
+        except Exception as error:
+            event["error"] = repr(error)
+            raise
+        finally:
+            controls.append(event)
+            (evidence / "http-controls.json").write_text(json.dumps(controls, indent=2) + "\n")
+
+    def snapshot(name=None):
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m07-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m07-ui.xml")
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
+    def rename(previous, title):
+        tap("Edit " + previous)
+        field = find("Edit item title")
+        tap("Edit item title")
+        adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len(field.get("text", ""))))
+        adb("shell", "input", "text", title.replace(" ", "%s"))
+        adb("shell", "input", "keyevent", "KEYCODE_BACK")
+        tap("Save title")
+        find("Local storage ready")
+        find(title, "text")
+
+    def launch(identity):
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--es", inputs["fixedLaunchIdentityExtra"], identity)
+        find("Local storage ready")
+
+    def read_database(path):
+        with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
+            db.row_factory = sqlite3.Row
+            assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+            schema = db.execute("PRAGMA user_version").fetchone()[0]
+            assert schema == (4 if args.baseline else 5), schema
+            items = [{"id": row["id"], "title": row["title"], "completed": bool(row["completed"]), "version": row["version"], "updatedAt": row["updated_at"]}
+                     for row in db.execute("SELECT * FROM items ORDER BY id")]
+            raw_pending = [dict(row) for row in db.execute("SELECT * FROM pending_mutations ORDER BY sequence")]
+            pending = []
+            for row in raw_pending:
+                intent = {"sequence": row["sequence"], "kind": row["kind"], "itemId": row["item_id"],
+                          "payload": None if row["payload"] is None else json.loads(row["payload"]),
+                          "clientMutationId": row["client_mutation_id"], "payloadHash": row["payload_hash"], "terminalError": row["terminal_error"]}
+                if not args.baseline:
+                    intent["dispatched"] = bool(row["dispatched"])
+                pending.append(intent)
+            conflicts = None if args.baseline else [{"intent": json.loads(row["intent"]), "reason": row["reason"],
+                "item": None if row["item"] is None else json.loads(row["item"]),
+                "tombstone": None if row["tombstone"] is None else json.loads(row["tombstone"])}
+                for row in db.execute("SELECT * FROM mutation_conflicts ORDER BY rowid")]
+            versions = None if args.baseline else [{"id": row["id"], "version": row["version"],
+                "updatedAt": row["updated_at"], "deleted": bool(row["deleted"])} for row in db.execute("SELECT * FROM remote_versions ORDER BY id")]
+            metadata = dict(db.execute("SELECT * FROM sync_metadata WHERE singleton=1").fetchone())
+            return {"schema_version": schema, "items": items, "pending": pending, "pendingSqlRows": raw_pending,
+                    "conflicts": conflicts, "remote_versions": versions, "sync_metadata": metadata,
+                    "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+
+    def database(name):
+        files = adb("shell", "run-as", PACKAGE, "ls", "files").splitlines()
+        path = evidence / f"{name}.db"
+        for suffix in ("", "-wal", "-shm"):
+            if "items.db" + suffix in files:
+                Path(str(path) + suffix).write_bytes(adb("exec-out", "run-as", PACKAGE, "cat", f"files/items.db{suffix}", binary=True))
+        value = read_database(path)
+        (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
+        return value
+
+    def network_state():
+        return {key: adb("shell", "settings", "get", "global", key)
+                for key in ("airplane_mode_on", "wifi_on", "mobile_data")}
+
+    def set_network(expected, name):
+        adb("shell", "cmd", "connectivity", "airplane-mode", "enable" if expected["airplane_mode_on"] == "1" else "disable")
+        adb("shell", "svc", "wifi", "enable" if expected["wifi_on"] == "1" else "disable")
+        adb("shell", "svc", "data", "enable" if expected["mobile_data"] == "1" else "disable")
+        deadline = time.monotonic() + 10
+        while True:
+            actual = network_state()
+            connectivity = adb("shell", "dumpsys", "connectivity")
+            no_network = "Active default network: none" in connectivity
+            if actual == expected and (no_network if expected == OFFLINE else not no_network):
+                break
+            assert time.monotonic() < deadline, f"Network did not reach {expected}: {actual}"
+            time.sleep(0.1)
+        (evidence / f"{name}-connectivity.txt").write_text(connectivity)
+        return actual
+
+    def substitute_baseline_identity(case, state):
+        # Only this metadata column changes, after the real offline UI edit and
+        # before any dispatch. Never alter the payload, hash, Item or allocator.
+        name = case["case"]
+        original = evidence / f"{name}-before-identity.db"
+        replacement = evidence / f"{name}-identity-setup.db"
+        with sqlite3.connect(f"file:{original}?mode=ro", uri=True) as source, sqlite3.connect(replacement) as db:
+            source.backup(db)
+            db.execute("PRAGMA journal_mode=DELETE")
+            changed = db.execute(inputs["baselineIdentitySetupSql"], (case["clientMutationId"], 1)).rowcount
+            assert changed == 1
+            db.commit()
+        expected = copy.deepcopy(state)
+        expected["pending"][0]["clientMutationId"] = case["clientMutationId"]
+        expected["pendingSqlRows"][0]["client_mutation_id"] = case["clientMutationId"]
+        assert read_database(replacement) == expected
+        record = {"sql": inputs["baselineIdentitySetupSql"], "parameters": [case["clientMutationId"], 1],
+                  "before": state, "after": expected, "databaseSha256": hashlib.sha256(replacement.read_bytes()).hexdigest()}
+        (evidence / f"{name}-identity-substitution.json").write_text(json.dumps(record, indent=2) + "\n")
+        replacement.chmod(0o644)
+        target = f"/data/local/tmp/mse-m07-{name}.db"
+        adb("push", str(replacement), target)
+        adb("shell", "run-as", PACKAGE, "rm", "-f", "files/items.db-wal", "files/items.db-shm")
+        adb("shell", "run-as", PACKAGE, "cp", target, "files/items.db")
+        adb("shell", "rm", target)
+        return expected
+
+    def get_event(items, tombstones):
+        return {"method": "GET", "path": "/items", "body": None, "status": 200,
+                "response": {"items": items, **({"tombstones": tombstones} if tombstones else {})}}
+
+    def expected_trace(case):
+        trace = [get_event([inputs["seed"]], [])]
+        if args.baseline:
+            outcome = case["baselineResult"]
+            response = {"error": "Item not found"} if case["case"] == "C" else (
+                {"deletedId": "conflict-001"} if case["case"] == "B" else {"item": outcome["items"][0]})
+            envelope = case["baselineEnvelope"]
+            trace.append({"method": envelope["method"], "path": envelope["path"], "body": envelope["payload"],
+                          "status": outcome["status"], "response": response})
+            if case["case"] != "C":
+                trace.append(get_event(outcome["items"], outcome["tombstones"]))
+            return trace
+        envelope = case["envelope"]
+        trace.append({"method": envelope["method"], "path": envelope["path"], "body": envelope["payload"],
+                      "status": 409, "response": case["conflictResponse"]})
+        trace.extend([get_event(case["canonicalItems"], case["canonicalTombstones"])] * 2)
+        if case["case"] == "A":
+            explicit = inputs["explicitEnvelope"]
+            trace.append({"method": explicit["method"], "path": explicit["path"], "body": explicit["payload"],
+                          "status": 200, "response": {"item": inputs["explicitItem"]}})
+            trace.append(get_event([inputs["explicitItem"]], []))
+        return trace
+
+    fixture = None
+    original_network = None
+    fixture_log = (evidence / "fixture.log").open("w")
+    try:
+        result["fixture_command"] = [args.node, "fixture/server.cjs"]
+        fixture = subprocess.Popen(result["fixture_command"], cwd=project, stdout=fixture_log, stderr=subprocess.STDOUT)
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
+        original_network = network_state()
+        assert original_network == {"airplane_mode_on": "0", "wifi_on": "1", "mobile_data": "1"}
+        result["network_before"] = set_network(original_network, "initial")
+        adb("install", "-r", args.apk)
+        adb("shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("shell", "wm", "dismiss-keyguard")
+        for case in inputs["cases"]:
+            name = case["case"]
+            current = {"case": name, "status": "RUNNING"}
+            result["cases"].append(current)
+            reset = remote("/__m07-reset", {})
+            assert reset == {"items": [inputs["seed"]], "nextTimestamp": 1700000501000, "requests": [], "delayMs": 0}
+            adb("shell", "am", "force-stop", PACKAGE)
+            assert adb("shell", "pm", "clear", PACKAGE) == "Success"
+            launch(case["clientMutationId"])
+            tap("Synchronize")
+            find("Sync status: fresh")
+            find("Local storage ready")
+            assert database(name + "-seed")["items"] == [inputs["seed"]]
+            set_network(OFFLINE, name + "-offline")
+            if case["kind"] == "delete":
+                tap("Delete Initial")
+                find("Local storage ready")
+                find("Item count: 0")
+            else:
+                rename("Initial", "Local attempt")
+            find("Pending uploads: 1")
+            current["setup_pid_before"] = adb("shell", "pidof", PACKAGE)
+            adb("shell", "am", "force-stop", PACKAGE)
+            assert not adb("shell", "pidof", PACKAGE, check=False)
+            before = database(name + "-before-identity")
+            assert len(before["pending"]) == 1
+            envelope = case["baselineEnvelope" if args.baseline else "envelope"]
+            assert before["pending"][0]["payload"] == envelope["payload"]
+            assert before["pending"][0]["payloadHash"] == envelope["payloadHash"]
+            if args.baseline:
+                before = substitute_baseline_identity(case, before)
+                current["identity_setup_last_command_index"] = len(commands)
+            else:
+                assert before["pending"][0] == {**case["conflictEvidence"]["intent"], "dispatched": False}
+            launch(case["clientMutationId"])
+            current["setup_pid_after"] = adb("shell", "pidof", PACKAGE)
+            assert current["setup_pid_after"] != current["setup_pid_before"]
+            current["before_dispatch"] = database(name + "-before-dispatch")
+            assert current["before_dispatch"] == before
+            assert network_state() == OFFLINE
+            assert remote()["requests"] == [get_event([inputs["seed"]], [])]
+            remote(case["remoteControl"]["path"], case["remoteControl"]["body"])
+            current["remote_before_dispatch"] = remote()
+            assert current["remote_before_dispatch"]["items"] == case["canonicalItems"]
+            assert current["remote_before_dispatch"].get("tombstones", []) == case["canonicalTombstones"]
+            current["first_dispatch_command_index"] = len(commands)
+            set_network(original_network, name + "-reconnected")
+            tap("Synchronize")
+            find("Sync status: error" if args.baseline and name == "C" else "Sync status: fresh")
+            find("Local storage ready")
+            current["after_dispatch"] = database(name + "-after-dispatch")
+            first = remote("/__mutation-state")
+            current["first_request_evidence"] = first
+            assert len(first["attempts"]) == 1
+            attempt = first["attempts"][0]
+            assert attempt["wireBody"] == envelope["wireBody"]
+            assert attempt["declaredHash"] == attempt["actualHash"] == envelope["payloadHash"]
+            if args.baseline:
+                outcome = case["baselineResult"]
+                assert attempt["status"] == outcome["status"] and first["appliedCount"] == outcome["appliedCount"]
+                assert len(current["after_dispatch"]["pending"]) == outcome["pendingCount"]
+                if name == "C":
+                    assert current["after_dispatch"] == current["before_dispatch"]
+                else:
+                    assert current["after_dispatch"]["items"] == outcome["items"]
+                assert current["after_dispatch"]["conflicts"] is None
+                assert not any(node.get("content-desc", "").startswith("Conflicts preserved:") for node in snapshot().iter("node"))
+            else:
+                assert attempt["status"] == 409 and attempt["response"] == case["conflictResponse"]
+                assert first["appliedCount"] == first["duplicateCount"] == 0
+                assert current["after_dispatch"]["items"] == case["canonicalItems"]
+                assert current["after_dispatch"]["pending"] == []
+                assert current["after_dispatch"]["conflicts"] == [case["conflictEvidence"]]
+                assert current["after_dispatch"]["sync_metadata"]["last_acknowledgement"] is None
+                assert current["after_dispatch"]["remote_versions"] == [{"id": "conflict-001", "version": 2, "updatedAt": 1700000500000, "deleted": name == "C"}]
+                find("Pending uploads: 0")
+                find("Conflicts preserved: 1")
+                current["conflict_death_boundary_start"] = len(commands)
+                current["conflict_pid_before"] = adb("shell", "pidof", PACKAGE)
+                adb("shell", "am", "force-stop", PACKAGE)
+                assert not adb("shell", "pidof", PACKAGE, check=False)
+                assert database(name + "-conflict-dead") == current["after_dispatch"]
+                launch(inputs["explicitEnvelope"]["clientMutationId"] if name == "A" else case["clientMutationId"])
+                current["conflict_pid_after"] = adb("shell", "pidof", PACKAGE)
+                assert current["conflict_pid_after"] != current["conflict_pid_before"]
+                assert database(name + "-conflict-reopened") == current["after_dispatch"]
+                current["conflict_death_boundary_end"] = len(commands)
+                assert remote("/__mutation-state") == first
+                find("Conflicts preserved: 1")
+                tap("Synchronize")
+                find("Sync status: fresh")
+                find("Local storage ready")
+                assert remote("/__mutation-state") == first, "An ordinary drain retried conflict evidence"
+                assert database(name + "-no-retry")["conflicts"] == [case["conflictEvidence"]]
+                if name == "A":
+                    rename("Remote winner", "Explicit edit")
+                    fresh = inputs["explicitEnvelope"]
+                    explicit_before = database("A-explicit-before-send")
+                    assert explicit_before["pending"] == [{"sequence": 2, "kind": "rename", "itemId": "conflict-001",
+                        "payload": fresh["payload"], "clientMutationId": fresh["clientMutationId"], "payloadHash": fresh["payloadHash"],
+                        "terminalError": None, "dispatched": False}]
+                    tap("Synchronize")
+                    find("Sync status: fresh")
+                    find("Local storage ready")
+                    explicit_after = database("A-explicit-after-send")
+                    assert explicit_after["items"] == [inputs["explicitItem"]]
+                    assert explicit_after["pending"] == [] and explicit_after["conflicts"] == [case["conflictEvidence"]]
+                    ack = json.loads(explicit_after["sync_metadata"]["last_acknowledgement"])
+                    assert ack == {"clientMutationId": fresh["clientMutationId"], "payloadHash": fresh["payloadHash"], "status": 200, "result": {"item": inputs["explicitItem"]}}
+                    current["explicit_edit"] = explicit_after
+            current["remote_final"] = remote()
+            current["mutation_evidence"] = remote("/__mutation-state")
+            assert current["remote_final"]["requests"] == expected_trace(case)
+            if args.baseline:
+                assert current["remote_final"]["items"] == case["baselineResult"]["items"]
+                assert current["remote_final"].get("tombstones", []) == case["baselineResult"]["tombstones"]
+                assert current["remote_final"]["nextTimestamp"] == case["baselineResult"]["nextTimestamp"]
+            else:
+                assert current["remote_final"]["items"] == ([inputs["explicitItem"]] if name == "A" else case["canonicalItems"])
+                assert current["remote_final"].get("tombstones", []) == case["canonicalTombstones"]
+                assert current["mutation_evidence"]["appliedCount"] == (1 if name == "A" else 0)
+                assert current["mutation_evidence"]["duplicateCount"] == 0
+            snapshot(name + "-final")
+            current["status"] = "BASELINE_LIMIT_REPRODUCED" if args.baseline else "PASS"
+            assert os.getpid() == result["host_pid"]
+            save()
+        result["status"] = "BASELINE_LIMIT_REPRODUCED" if args.baseline else "PASS"
+    except Exception as error:
+        result["status"], result["error"] = "FAIL", repr(error)
+    finally:
+        if original_network is not None:
+            try:
+                result["network_after"] = set_network(original_network, "restored")
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
+                result["status"], result["fixture_cleanup_error"] = "FAIL", "Owned fixture required forced cleanup"
+            if result["fixture_exit"] != 0:
+                result["status"], result["fixture_cleanup_error"] = "FAIL", "Owned fixture did not exit cleanly"
+        fixture_log.close()
+        result["adb_command_count"] = len(commands)
+        save()
+        print(json.dumps(result, indent=2), flush=True)
+    if result["status"] == "FAIL":
+        raise AssertionError(result.get("error", result.get("cleanup_error", "M07 did not complete")))
+
+
+if __name__ == "__main__":
+    main()
diff --git a/verification/M07-inputs.json b/verification/M07-inputs.json
new file mode 100644
index 0000000..dfec883
--- /dev/null
+++ b/verification/M07-inputs.json
@@ -0,0 +1,483 @@
+{
+  "profile": "phase-1",
+  "thread": "M07",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "f1107d4f8f667dc4d440358181b4ff92f0b9e030",
+  "scenarioSha256": "9576755a0f5ae5b6a573932b1fe6dad4fb24b853950678301a12f3bc2e9314e3",
+  "seed": {
+    "id": "conflict-001",
+    "title": "Initial",
+    "completed": false,
+    "version": 1,
+    "updatedAt": 1700000000000
+  },
+  "remoteWinner": {
+    "id": "conflict-001",
+    "title": "Remote winner",
+    "completed": false,
+    "version": 2,
+    "updatedAt": 1700000500000
+  },
+  "remoteTombstone": {
+    "id": "conflict-001",
+    "version": 2,
+    "updatedAt": 1700000500000,
+    "deleted": true
+  },
+  "explicitItem": {
+    "id": "conflict-001",
+    "title": "Explicit edit",
+    "completed": false,
+    "version": 3,
+    "updatedAt": 1700000501000
+  },
+  "explicitEnvelope": {
+    "method": "PATCH",
+    "path": "/items/conflict-001",
+    "payload": {
+      "title": "Explicit edit",
+      "baseVersion": 2
+    },
+    "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":2,\"title\":\"Explicit edit\"}}",
+    "payloadHash": "856ae7060b3fce24de5fd23993382779be3977ca189d3e6104a7f533a67f128d",
+    "clientMutationId": "m07-explicit-001",
+    "wireBody": {
+      "title": "Explicit edit",
+      "baseVersion": 2,
+      "clientMutationId": "m07-explicit-001",
+      "payloadHash": "856ae7060b3fce24de5fd23993382779be3977ca189d3e6104a7f533a67f128d"
+    }
+  },
+  "baselineIdentitySetupSql": "UPDATE pending_mutations SET client_mutation_id = ? WHERE sequence = ?",
+  "baselineIdentitySetupScope": "After real offline UI mutation and force-stop, before any dispatch: substitute only client_mutation_id. Capture native SQL values before/after and retain original payload TEXT/hash/Item state exactly. No post-send envelope edits.",
+  "fixedLaunchIdentityExtra": "m07MutationIdentity",
+  "delayMs": 0,
+  "nextAcceptedTimestamp": 1700000501000,
+  "cases": [
+    {
+      "case": "A",
+      "kind": "rename",
+      "clientMutationId": "m07-update-001",
+      "envelope": {
+        "method": "PATCH",
+        "path": "/items/conflict-001",
+        "payload": {
+          "title": "Local attempt",
+          "baseVersion": 1
+        },
+        "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":1,\"title\":\"Local attempt\"}}",
+        "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d",
+        "clientMutationId": "m07-update-001",
+        "wireBody": {
+          "title": "Local attempt",
+          "baseVersion": 1,
+          "clientMutationId": "m07-update-001",
+          "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d"
+        }
+      },
+      "baselineEnvelope": {
+        "method": "PATCH",
+        "path": "/items/conflict-001",
+        "payload": {
+          "title": "Local attempt"
+        },
+        "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"title\":\"Local attempt\"}}",
+        "payloadHash": "56f0b93d786790567c73d140be469e1666eb01a87a6030e0d8c9e24ec620d55c",
+        "clientMutationId": "m07-update-001",
+        "wireBody": {
+          "title": "Local attempt",
+          "clientMutationId": "m07-update-001",
+          "payloadHash": "56f0b93d786790567c73d140be469e1666eb01a87a6030e0d8c9e24ec620d55c"
+        }
+      },
+      "remoteControl": {
+        "path": "/__remote-change",
+        "body": {
+          "id": "conflict-001",
+          "updatedAt": 1700000500000,
+          "title": "Remote winner"
+        }
+      },
+      "conflictResponse": {
+        "error": "version_conflict",
+        "item": {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        },
+        "tombstone": null
+      },
+      "conflictEvidence": {
+        "intent": {
+          "sequence": 1,
+          "kind": "rename",
+          "itemId": "conflict-001",
+          "payload": {
+            "title": "Local attempt",
+            "baseVersion": 1
+          },
+          "clientMutationId": "m07-update-001",
+          "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d",
+          "terminalError": null,
+          "dispatched": true
+        },
+        "reason": "version_conflict",
+        "item": {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        },
+        "tombstone": null
+      },
+      "canonicalItems": [
+        {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        }
+      ],
+      "canonicalTombstones": [],
+      "baselineResult": {
+        "status": 200,
+        "items": [
+          {
+            "id": "conflict-001",
+            "title": "Local attempt",
+            "completed": false,
+            "version": 3,
+            "updatedAt": 1700000501000
+          }
+        ],
+        "tombstones": [],
+        "pendingCount": 0,
+        "appliedCount": 1,
+        "nextTimestamp": 1700000502000
+      }
+    },
+    {
+      "case": "B",
+      "kind": "delete",
+      "clientMutationId": "m07-delete-001",
+      "envelope": {
+        "method": "DELETE",
+        "path": "/items/conflict-001",
+        "payload": {
+          "baseVersion": 1
+        },
+        "canonical": "{\"method\":\"DELETE\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":1}}",
+        "payloadHash": "dcdc12c2281f2619b08cf53b7fb3aebf5c1d9fa3e0273b2857868ad75c08c736",
+        "clientMutationId": "m07-delete-001",
+        "wireBody": {
+          "baseVersion": 1,
+          "clientMutationId": "m07-delete-001",
+          "payloadHash": "dcdc12c2281f2619b08cf53b7fb3aebf5c1d9fa3e0273b2857868ad75c08c736"
+        }
+      },
+      "baselineEnvelope": {
+        "method": "DELETE",
+        "path": "/items/conflict-001",
+        "payload": null,
+        "canonical": "{\"method\":\"DELETE\",\"path\":\"/items/conflict-001\",\"payload\":null}",
+        "payloadHash": "f6e682c882d9d2aeeb05cc05f683e951cbf4c6d2a824462fc2a8e4962b9b10f7",
+        "clientMutationId": "m07-delete-001",
+        "wireBody": {
+          "clientMutationId": "m07-delete-001",
+          "payloadHash": "f6e682c882d9d2aeeb05cc05f683e951cbf4c6d2a824462fc2a8e4962b9b10f7"
+        }
+      },
+      "remoteControl": {
+        "path": "/__remote-change",
+        "body": {
+          "id": "conflict-001",
+          "updatedAt": 1700000500000,
+          "title": "Remote winner"
+        }
+      },
+      "conflictResponse": {
+        "error": "version_conflict",
+        "item": {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        },
+        "tombstone": null
+      },
+      "conflictEvidence": {
+        "intent": {
+          "sequence": 1,
+          "kind": "delete",
+          "itemId": "conflict-001",
+          "payload": {
+            "baseVersion": 1
+          },
+          "clientMutationId": "m07-delete-001",
+          "payloadHash": "dcdc12c2281f2619b08cf53b7fb3aebf5c1d9fa3e0273b2857868ad75c08c736",
+          "terminalError": null,
+          "dispatched": true
+        },
+        "reason": "version_conflict",
+        "item": {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        },
+        "tombstone": null
+      },
+      "canonicalItems": [
+        {
+          "id": "conflict-001",
+          "title": "Remote winner",
+          "completed": false,
+          "version": 2,
+          "updatedAt": 1700000500000
+        }
+      ],
+      "canonicalTombstones": [],
+      "baselineResult": {
+        "status": 200,
+        "items": [],
+        "tombstones": [
+          {
+            "id": "conflict-001",
+            "version": 3,
+            "updatedAt": 1700000501000,
+            "deleted": true
+          }
+        ],
+        "pendingCount": 0,
+        "appliedCount": 1,
+        "nextTimestamp": 1700000502000
+      }
+    },
+    {
+      "case": "C",
+      "kind": "rename",
+      "clientMutationId": "m07-deleted-001",
+      "envelope": {
+        "method": "PATCH",
+        "path": "/items/conflict-001",
+        "payload": {
+          "title": "Local attempt",
+          "baseVersion": 1
+        },
+        "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":1,\"title\":\"Local attempt\"}}",
+        "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d",
+        "clientMutationId": "m07-deleted-001",
+        "wireBody": {
+          "title": "Local attempt",
+          "baseVersion": 1,
+          "clientMutationId": "m07-deleted-001",
+          "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d"
+        }
+      },
+      "baselineEnvelope": {
+        "method": "PATCH",
+        "path": "/items/conflict-001",
+        "payload": {
+          "title": "Local attempt"
+        },
+        "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"title\":\"Local attempt\"}}",
+        "payloadHash": "56f0b93d786790567c73d140be469e1666eb01a87a6030e0d8c9e24ec620d55c",
+        "clientMutationId": "m07-deleted-001",
+        "wireBody": {
+          "title": "Local attempt",
+          "clientMutationId": "m07-deleted-001",
+          "payloadHash": "56f0b93d786790567c73d140be469e1666eb01a87a6030e0d8c9e24ec620d55c"
+        }
+      },
+      "remoteControl": {
+        "path": "/__remote-delete",
+        "body": {
+          "id": "conflict-001",
+          "updatedAt": 1700000500000
+        }
+      },
+      "conflictResponse": {
+        "error": "version_conflict",
+        "item": null,
+        "tombstone": {
+          "id": "conflict-001",
+          "version": 2,
+          "updatedAt": 1700000500000,
+          "deleted": true
+        }
+      },
+      "conflictEvidence": {
+        "intent": {
+          "sequence": 1,
+          "kind": "rename",
+          "itemId": "conflict-001",
+          "payload": {
+            "title": "Local attempt",
+            "baseVersion": 1
+          },
+          "clientMutationId": "m07-deleted-001",
+          "payloadHash": "3a20776167a2f7bcd285fce59e9b6ce1a855ad169b067726ce2016102d1fcd8d",
+          "terminalError": null,
+          "dispatched": true
+        },
+        "reason": "version_conflict",
+        "item": null,
+        "tombstone": {
+          "id": "conflict-001",
+          "version": 2,
+          "updatedAt": 1700000500000,
+          "deleted": true
+        }
+      },
+      "canonicalItems": [],
+      "canonicalTombstones": [
+        {
+          "id": "conflict-001",
+          "version": 2,
+          "updatedAt": 1700000500000,
+          "deleted": true
+        }
+      ],
+      "baselineResult": {
+        "status": 404,
+        "items": [],
+        "tombstones": [
+          {
+            "id": "conflict-001",
+            "version": 2,
+            "updatedAt": 1700000500000,
+            "deleted": true
+          }
+        ],
+        "pendingCount": 1,
+        "appliedCount": 0,
+        "nextTimestamp": 1700000501000
+      }
+    }
+  ],
+  "successorRegression": {
+    "firstIdentity": "m07-own-001",
+    "successorIdentity": "m07-successor-001",
+    "first": {
+      "method": "PATCH",
+      "path": "/items/conflict-001",
+      "payload": {
+        "title": "Own predecessor",
+        "baseVersion": 1
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":1,\"title\":\"Own predecessor\"}}",
+      "payloadHash": "a5785ed56e536346a053730a07a70a0af1ef7c08ac6022946b4cf40b2d3910c3",
+      "clientMutationId": "m07-own-001",
+      "wireBody": {
+        "title": "Own predecessor",
+        "baseVersion": 1,
+        "clientMutationId": "m07-own-001",
+        "payloadHash": "a5785ed56e536346a053730a07a70a0af1ef7c08ac6022946b4cf40b2d3910c3"
+      }
+    },
+    "initialSuccessor": {
+      "method": "PATCH",
+      "path": "/items/conflict-001",
+      "payload": {
+        "completed": true,
+        "baseVersion": 1
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":1,\"completed\":true}}",
+      "payloadHash": "6ae53d4afb30d34507b34f3c6b3f3a0a684aa5aa94f6128326f89e355ee48e62",
+      "clientMutationId": "m07-successor-001",
+      "wireBody": {
+        "completed": true,
+        "baseVersion": 1,
+        "clientMutationId": "m07-successor-001",
+        "payloadHash": "6ae53d4afb30d34507b34f3c6b3f3a0a684aa5aa94f6128326f89e355ee48e62"
+      }
+    },
+    "preparedSuccessor": {
+      "method": "PATCH",
+      "path": "/items/conflict-001",
+      "payload": {
+        "completed": true,
+        "baseVersion": 2
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":2,\"completed\":true}}",
+      "payloadHash": "678e382d7620cc2e02a2c42601f8b860d14e32897978b500e1b3bddc836d6232",
+      "clientMutationId": "m07-successor-001",
+      "wireBody": {
+        "completed": true,
+        "baseVersion": 2,
+        "clientMutationId": "m07-successor-001",
+        "payloadHash": "678e382d7620cc2e02a2c42601f8b860d14e32897978b500e1b3bddc836d6232"
+      }
+    },
+    "baseOnlyCollision": {
+      "method": "PATCH",
+      "path": "/items/conflict-001",
+      "payload": {
+        "title": "Own predecessor",
+        "baseVersion": 2
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"baseVersion\":2,\"title\":\"Own predecessor\"}}",
+      "payloadHash": "b4671bd40877458d7d51d00723307f1ebf7385932fe1df7d344c64f3047edf96",
+      "clientMutationId": "m07-own-001",
+      "wireBody": {
+        "title": "Own predecessor",
+        "baseVersion": 2,
+        "clientMutationId": "m07-own-001",
+        "payloadHash": "b4671bd40877458d7d51d00723307f1ebf7385932fe1df7d344c64f3047edf96"
+      }
+    },
+    "firstResult": {
+      "id": "conflict-001",
+      "title": "Own predecessor",
+      "completed": false,
+      "version": 2,
+      "updatedAt": 1700000501000
+    },
+    "successorResult": {
+      "id": "conflict-001",
+      "title": "Own predecessor",
+      "completed": true,
+      "version": 3,
+      "updatedAt": 1700000502000
+    },
+    "externalAfterLoss": {
+      "id": "conflict-001",
+      "title": "External after loss",
+      "completed": true,
+      "version": 4,
+      "updatedAt": 1700000503000
+    },
+    "externalBeforeSuccessor": {
+      "id": "conflict-001",
+      "title": "External before successor",
+      "completed": false,
+      "version": 3,
+      "updatedAt": 1700000501500
+    }
+  },
+  "legacyUpgrade": {
+    "identity": "m07-legacy-update-001",
+    "original": {
+      "method": "PATCH",
+      "path": "/items/conflict-001",
+      "payload": {
+        "title": "Legacy attempt"
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/conflict-001\",\"payload\":{\"title\":\"Legacy attempt\"}}",
+      "payloadHash": "789f5c0447150f6be990efbdf0d804d33be25102a7ab82b922296996ffb87462",
+      "clientMutationId": "m07-legacy-update-001",
+      "wireBody": {
+        "title": "Legacy attempt",
+        "clientMutationId": "m07-legacy-update-001",
+        "payloadHash": "789f5c0447150f6be990efbdf0d804d33be25102a7ab82b922296996ffb87462"
+      }
+    },
+    "reason": "unversioned_legacy"
+  }
+}
diff --git a/verification/M07.md b/verification/M07.md
new file mode 100644
index 0000000..b1f6577
--- /dev/null
+++ b/verification/M07.md
@@ -0,0 +1,17 @@
+# M07 verification — phase-1
+
+- Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: verified M06 `f1107d4f8f667dc4d440358181b4ff92f0b9e030`.
+- Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/`.
+
+## Initial baseline (attempt1): incomplete
+
+`run.py baseline-android-01 python3 scripts/verify_m07.py --baseline` used the exact frozen command in `baseline-android-01.command.json`: exit1, 193.915s. A reproduced stale update acceptance (local title became canonical version3); B reproduced stale delete acceptance (remote tombstone version3). Their native databases, identity-only setup, UI and HTTP evidence are in `baseline-android-01/`.
+
+C failed during its initial `am start -W` (60s timeout), before local seed, edit, identity setup or dispatch. C has no baseline acceptance result. The original adb helper did not record the timed-out command's partial stdout; it is unavailable and has not been reconstructed. `baseline-android-01/result.json`, `.command.json` and `.log` remain unchanged.
+
+Frozen support is preserved in `frozen-support/` and `frozen-inputs-before-baseline.json`: harness `a7bdb1fff1417c6a4903a8e1124c07230814ea75e1ea293978a4f8b8d963e063`, fixture `e8bac2fca47210b7d64ee7d7eb86154a4e9af146cfef91fe6974b9fe702e674d`, inputs `53c14f6580e9a33f5d5f15b23ecc8137f600eecf2553a88683498f97e7d7ac64`. All 49 other START files and the verified M06 APK (`f9ef2c2c384a21fda0da03243d1160097b0780d400d2084aee26844633dccb90`) are unchanged. No M07 product implementation is included.
+
+Cleanup in the original result: network restored to `0/1/1`, app PID absent, owned fixture exit0. Root's preserved `threads/evidence/phase-1/react-native/M07/main-launch-diagnostic-01/` identifies the old task172 removal callback killing new process14551 at 13:17:39.542. A fresh bounded harness repair uses repair1 of the hard maximum2; this count remains part of M07.
+
+Host-only audit `repair1-original-audit-01` passed (exit0): ten raw A/B database snapshots, ID-only substitution, wire hashes, side effects and UI; original evidence inventory retained at `repair1/original-evidence-sha256.json`.


