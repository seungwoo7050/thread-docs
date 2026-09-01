# M09 — Process Death 이후 복구

## `test(m09): freeze actual process-death startup baseline`

diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 0874176..2b9f645 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -135,6 +135,51 @@ async function control(path: string, body: unknown) {
   return response.json();
 }
 
+test('M09 fixture commits before holding ACK, serves controls, then replays the original receipt after drop', async () => {
+  const inputs = require('../verification/M09-inputs.json');
+  const scenario = inputs.cases[1];
+  await control('/__m09-reset', {case: 'B'});
+  await control('/__m09-arm', {clientMutationId: scenario.clientMutationId});
+  const options = {method: scenario.method, headers: {'Content-Type': 'application/json'}, body: JSON.stringify(scenario.wireBody)};
+  let settled = false;
+  const heldRequest = request(url + scenario.path, options).then(
+    response => {settled = true; return {response};},
+    error => {settled = true; return {error: String(error)};},
+  );
+  let hold: any;
+  try {
+    const deadline = Date.now() + 3000;
+    do {
+      hold = await (await request(url + '/__m09-hold', {method: 'GET', headers: {}})).json();
+      if (hold.phase === 'COMMITTED_HELD') {break;}
+      if (Date.now() >= deadline) {throw new Error('Commit barrier not reached');}
+      await new Promise(resolve => setTimeout(resolve, 10));
+    } while (true);
+    expect(settled).toBe(false);
+    expect(hold).toMatchObject({phase: 'COMMITTED_HELD', maxHoldMs: 30000, headersSent: false,
+      status: 200, response: {item: scenario.finalItem}, canonical: scenario.canonical, payloadHash: scenario.payloadHash});
+    expect(hold.deadlineAt - hold.committedAt).toBe(30000);
+    expect(fixture.state().items).toEqual([scenario.finalItem]);
+    expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 0, conflictCount: 0, hashRejectedCount: 0});
+    expect(fixture.mutationState().attempts).toEqual([expect.objectContaining({outcome: 'applied',
+      wireBody: scenario.wireBody, responseHeld: true, canonical: scenario.canonical})]);
+    const dropped = await control('/__m09-drop', {}) as {phase: string; endedAt: number; deadlineAt: number};
+    expect(dropped.phase).toBe('DROPPED');
+    expect(dropped.endedAt).toBeLessThan(dropped.deadlineAt);
+    expect(await heldRequest).toHaveProperty('error');
+    const replay = await request(url + scenario.path, options);
+    expect(replay.status).toBe(200);
+    expect(await replay.json()).toEqual({item: scenario.finalItem});
+    expect(fixture.state().items).toEqual([scenario.finalItem]);
+    expect(fixture.state().nextTimestamp).toBe(1700000601000);
+    expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 1, conflictCount: 0, hashRejectedCount: 0});
+    console.info('M09_COMMIT_HOLD_DROP_REPLAY', JSON.stringify({hold, dropped, state: fixture.state(), mutations: fixture.mutationState()}));
+  } finally {
+    fixture.reset(); // Also drops a held response if an assertion failed.
+    await heldRequest;
+  }
+});
+
 test('M04 frozen HTTP failure and reconnect retain persistent Items and successful-refresh time', async () => {
   let offline = false;
   const transport: JsonRequest = (address, options) => offline
diff --git a/fixture/server.cjs b/fixture/server.cjs
index 6f2d056..f74d062 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03–M07 fixture, not a backend service.
+// A disposable deterministic M03–M09 fixture, not a backend service.
 const http = require('node:http');
 const {createHash} = require('node:crypto');
 
@@ -26,9 +26,25 @@ function createFixture() {
   let dropFirstCreate;
   let dropMutationId;
   let tombstones;
+  let heldResponse = null;
+  let holdTimer = null;
+  let hold = {phase: 'IDLE', maxHoldMs: 30000};
+  function dropHeldResponse(phase) {
+    if (hold.phase !== 'COMMITTED_HELD') {return false;}
+    clearTimeout(holdTimer);
+    hold = {...hold, phase, endedAt: Date.now()};
+    heldResponse.destroy(); // Never deliver any acknowledgment bytes.
+    heldResponse = null;
+    holdTimer = null;
+    return true;
+  }
+  const holdState = () => ({...hold,
+    ...(heldResponse === null ? {} : {headersSent: heldResponse.headersSent, connectionClosed: heldResponse.destroyed})});
   const snapshot = () => [...items.values()].sort((a, b) => a.id.localeCompare(b.id));
   const deletedSnapshot = () => [...tombstones.values()].sort((a, b) => a.id.localeCompare(b.id));
   function reset() {
+    dropHeldResponse('RESET_DROPPED');
+    hold = {phase: 'IDLE', maxHoldMs: 30000};
     items = new Map(SEEDS.map(item => [item.id, {...item}]));
     nextTimestamp = 1700000100000;
     requests = [];
@@ -63,7 +79,19 @@ function createFixture() {
       }
       const responseDropped = Boolean(applied && ((dropFirstCreate && method === 'POST' && path === '/items' && body.id === 'crash-001')
         || (dropMutationId !== null && clientMutationId === dropMutationId)));
-      if (mutation) {mutationEvidence.attempts.push({...mutation, outcome, status, response: payload, responseDropped});}
+      const responseHeld = applied && hold.phase === 'ARMED' && clientMutationId === hold.clientMutationId;
+      if (mutation) {mutationEvidence.attempts.push({...mutation, outcome, status, response: payload, responseDropped,
+        ...(responseHeld ? {responseHeld: true} : {})});}
+      if (responseHeld) {
+        // Item, timestamp, receipt and applied evidence above are committed before
+        // this barrier is observable. Controls keep serving on separate requests.
+        const committedAt = Date.now();
+        hold = {...hold, phase: 'COMMITTED_HELD', committedAt, deadlineAt: committedAt + 30000,
+          status, response: payload, canonical: mutation.canonical, payloadHash: mutation.actualHash};
+        heldResponse = response;
+        holdTimer = setTimeout(() => dropHeldResponse('EXPIRED'), 30000);
+        return;
+      }
       if (responseDropped) {
         dropFirstCreate = false;
         dropMutationId = null;
@@ -86,6 +114,24 @@ function createFixture() {
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
       if (method === 'GET' && path === '/__mutation-state') {return reply(200, mutationEvidence);}
+      if (method === 'POST' && path === '/__m09-reset') {
+        if (body?.case !== 'A' && body?.case !== 'B') {return reply(400, {error: 'M09 case A or B required'});}
+        reset();
+        items.clear();
+        if (body.case === 'B') {items.set('death-001', {id: 'death-001', title: 'Initial', completed: false, version: 1, updatedAt: 1700000000000});}
+        nextTimestamp = 1700000600000;
+        return reply(200, {...state(), delayMs: 0});
+      }
+      if (method === 'POST' && path === '/__m09-arm') {
+        if (body?.clientMutationId !== 'm09-update-001' || hold.phase !== 'IDLE') {return reply(400, {error: 'Fresh M09 update barrier required'});}
+        hold = {phase: 'ARMED', clientMutationId: body.clientMutationId, maxHoldMs: 30000};
+        return reply(200, holdState());
+      }
+      if (method === 'GET' && path === '/__m09-hold') {return reply(200, holdState());}
+      if (method === 'POST' && path === '/__m09-drop') {
+        if (!dropHeldResponse('DROPPED')) {return reply(409, {error: 'No committed held response'});}
+        return reply(200, holdState());
+      }
       if (method === 'POST' && path === '/__m06-reset') {
         reset();
         items.clear();
diff --git a/scripts/verify_m09.py b/scripts/verify_m09.py
new file mode 100644
index 0000000..8d09bd5
--- /dev/null
+++ b/scripts/verify_m09.py
@@ -0,0 +1,459 @@
+#!/usr/bin/env python3
+"""Frozen M09 external process-death cases. Exclusive Android lease required.
+
+Baseline A uses the approved prelaunch schema5 input, not production create.
+Fixed A and both B renames use the UI. Nothing injects state after first launch.
+"""
+import argparse
+import hashlib
+import io
+import json
+import os
+from pathlib import Path
+import re
+import socket
+import sqlite3
+import subprocess
+import tarfile
+import time
+from urllib.error import HTTPError
+from urllib.request import Request, urlopen
+import xml.etree.ElementTree as ET
+from verify_m07 import package_in_live_activities
+
+PACKAGE = "com.mse.reactnative"
+URL = "http://127.0.0.1:18081"
+NETWORK_KEYS = ("airplane_mode_on", "wifi_on", "mobile_data")
+ONLINE = dict(zip(NETWORK_KEYS, ("0", "1", "1")))
+OFFLINE = dict(zip(NETWORK_KEYS, ("1", "0", "0")))
+TABLES = ("items", "pending_mutations", "mutation_conflicts", "remote_versions", "sync_metadata", "local_identity", "sqlite_sequence")
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
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs_path = root / "verification/M09-inputs.json"
+    inputs = json.loads(inputs_path.read_text())
+    sha = lambda path: hashlib.sha256(Path(path).read_bytes()).hexdigest()
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "hostPid": os.getpid(), "serial": args.serial,
+              "apkSha256": sha(args.apk), "inputsSha256": sha(inputs_path), "harnessSha256": sha(__file__),
+              "fixtureSha256": sha(root / "fixture/server.cjs"),
+              "resetPredicateSourceSha256": sha(root / "scripts/verify_m07.py"), "cases": []}
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(label, *parts, check=True, binary=False, timeout=60):
+        command = [args.adb, "-s", args.serial, *parts]
+        entry = {"label": label, "command": command, "timeoutSeconds": timeout, "startedAt": int(time.time() * 1000)}
+        started = time.monotonic()
+        try:
+            completed = subprocess.run(command, capture_output=True, timeout=timeout)
+            raw, err = completed.stdout, completed.stderr
+            entry["exit"] = completed.returncode
+        except subprocess.TimeoutExpired as error:
+            raw, err = error.stdout, error.stderr
+            entry.update(exit=None, error=repr(error))
+        for stream, data in (("stdout", raw), ("stderr", err)):
+            entry[stream] = None
+            if data is not None:
+                filename = f"adb-{len(commands):04d}-{label}.{stream}"
+                (evidence / filename).write_bytes(data)
+                entry[stream] = filename
+        entry.update(elapsedSeconds=time.monotonic() - started, endedAt=int(time.time() * 1000))
+        commands.append(entry)
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        assert entry["exit"] is not None, entry
+        if check:
+            assert entry["exit"] == 0, entry
+        return raw if binary else raw.decode(errors="replace").strip()
+
+    def remote(path="/__state", body=None):
+        event = {"method": "GET" if body is None else "POST", "path": path, "body": body,
+                 "startedAt": int(time.time() * 1000)}
+        try:
+            request = Request(URL + path, data=None if body is None else json.dumps(body).encode(),
+                              headers={"Content-Type": "application/json"}, method=event["method"])
+            try:
+                response = urlopen(request, timeout=3)
+            except HTTPError as error:
+                response = error
+            with response:
+                raw = response.read().decode()
+                event.update(status=response.status, headers=dict(response.headers), rawBody=raw, response=json.loads(raw))
+            assert event["status"] == 200, event
+            return event["response"]
+        except Exception as error:
+            event["error"] = repr(error)
+            raise
+        finally:
+            event["endedAt"] = int(time.time() * 1000)
+            controls.append(event)
+            (evidence / "http-controls.json").write_text(json.dumps(controls, indent=2) + "\n")
+
+    def network(label, expected=None):
+        if expected is not None:
+            adb(label + "-airplane", "shell", "cmd", "connectivity", "airplane-mode", "enable" if expected == OFFLINE else "disable")
+            adb(label + "-wifi", "shell", "svc", "wifi", "disable" if expected == OFFLINE else "enable")
+            adb(label + "-data", "shell", "svc", "data", "disable" if expected == OFFLINE else "enable")
+        deadline = time.monotonic() + 10
+        while True:
+            settings = {key: adb(label + "-" + key, "shell", "settings", "get", "global", key) for key in NETWORK_KEYS}
+            connectivity = adb(label + "-connectivity", "shell", "dumpsys", "connectivity")
+            no_network = "Active default network: none" in connectivity
+            if expected is None or (settings == expected and no_network == (expected == OFFLINE)):
+                assert no_network == (settings == OFFLINE)
+                return settings
+            assert time.monotonic() < deadline, f"Network failed to reach {expected}: {settings}"
+            time.sleep(0.1)
+
+    def quiet_reset(label):
+        # Reuse the main-verified API34 live-record parser, not historic pointers.
+        started, quiet = time.monotonic(), None
+        observations = []
+        result.setdefault("resetObservations", {})[label] = observations
+        while True:
+            remaining = inputs["resetTimeoutSeconds"] - (time.monotonic() - started)
+            assert remaining > 0, "Activity teardown did not settle within 10s"
+            pid = adb(label + "-pid", "shell", "pidof", PACKAGE, check=False, timeout=remaining)
+            assert commands[-1]["exit"] in (0, 1)
+            remaining = inputs["resetTimeoutSeconds"] - (time.monotonic() - started)
+            assert remaining > 0
+            activities = adb(label + "-activities", "shell", "dumpsys", "activity", "activities", timeout=remaining)
+            present = package_in_live_activities(activities)
+            now = time.monotonic()
+            observations.append({"elapsedSeconds": now - started, "pid": pid, "liveActivity": present,
+                                 "pidCommandIndex": len(commands) - 2, "activityCommandIndex": len(commands) - 1})
+            save()
+            assert now - started < inputs["resetTimeoutSeconds"]
+            if not pid and not present:
+                quiet = now if quiet is None else quiet
+                if now - quiet >= inputs["resetQuietSeconds"]:
+                    return
+            else:
+                quiet = None
+            time.sleep(0.1)
+
+    def snapshot(name=None):
+        adb("ui-dump", "shell", "uiautomator", "dump", "/sdcard/mse-m09-ui.xml")
+        xml = adb("ui-read", "exec-out", "cat", "/sdcard/mse-m09-ui.xml")
+        if name:
+            (evidence / (name + ".xml")).write_text(xml)
+            (evidence / (name + ".png")).write_bytes(adb("screenshot", "exec-out", "screencap", "-p", binary=True))
+        return ET.fromstring(xml)
+
+    def find(label, attribute="content-desc"):
+        deadline = time.monotonic() + inputs["uiTimeoutSeconds"]
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
+        adb("tap-" + label.replace(" ", "-"), "shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
+
+    def launch(label, extras=()):
+        adb(label, "shell", "am", "start", "-W", "-n", PACKAGE + "/.MainActivity", *extras)
+        pid = adb(label + "-pid", "shell", "pidof", PACKAGE)
+        assert re.fullmatch(r"\d+", pid), pid
+        return pid
+
+    def read_database(path):
+        before_hash = sha(path)
+        with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
+            db.row_factory = sqlite3.Row
+            assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+            value = {"schemaVersion": db.execute("PRAGMA user_version").fetchone()[0]}
+            value.update({table: [dict(row) for row in db.execute("SELECT * FROM " + table + " ORDER BY rowid")] for table in TABLES})
+        assert value["schemaVersion"] == inputs["schemaVersion"] and sha(path) == before_hash
+        return value
+
+    def database(name, deadline=None):
+        def remaining():
+            left = 60 if deadline is None else (deadline - int(time.time() * 1000)) / 1000
+            assert left > 0, "Held ACK deadline elapsed before native evidence/kill"
+            return min(left, 60)
+        files = adb(name + "-files", "shell", "run-as", PACKAGE, "ls", "files", timeout=remaining()).splitlines()
+        names = ["files/items.db" + suffix for suffix in ("", "-wal", "-shm") if "items.db" + suffix in files]
+        assert "files/items.db" in names
+        raw = adb(name + "-native", "exec-out", "run-as", PACKAGE, "tar", "-cf", "-", *names, binary=True, timeout=remaining())
+        archive = evidence / (name + ".tar")
+        archive.write_bytes(raw)
+        path = evidence / (name + ".db")
+        with tarfile.open(fileobj=io.BytesIO(raw)) as saved:
+            for member in saved:
+                assert member.isfile() and member.name in names
+                suffix = member.name.removeprefix("files/items.db")
+                Path(str(path) + suffix).write_bytes(saved.extractfile(member).read())
+        value = read_database(path)
+        (evidence / (name + ".json")).write_text(json.dumps(value, indent=2) + "\n")
+        result.setdefault("nativeDatabases", []).append({"name": name, "archiveSha256": sha(archive), "databaseSha256": sha(path)})
+        return value
+
+    def seed(case):
+        # Approved disposable input only: after OS offline control for A, before
+        # the FIRST launch. This does not claim to invoke production mutation.
+        name = case["case"]
+        path = evidence / (name + "-setup.db")
+        statements = case["seedStatements"] + (case["baselineOnlyStatements"] if args.baseline else [])
+        with sqlite3.connect(path) as db:
+            db.execute("PRAGMA journal_mode=DELETE")
+            db.execute("BEGIN IMMEDIATE")
+            for sql in inputs["schemaSql"]:
+                db.execute(sql)
+            for statement in statements:
+                db.execute(statement["sql"], statement["params"])
+            db.commit()
+        path.chmod(0o644)
+        (evidence / (name + "-setup-sql.json")).write_text(json.dumps({"schemaSql": inputs["schemaSql"], "statements": statements}, indent=2) + "\n")
+        expected = read_database(path)
+        temporary = "/data/local/tmp/mse-m09-seed.db"
+        adb(name + "-seed-push", "push", str(path), temporary)
+        adb(name + "-seed-directory", "shell", "run-as", PACKAGE, "mkdir", "-p", "files")
+        adb(name + "-seed-copy", "shell", "run-as", PACKAGE, "cp", temporary, "files/items.db")
+        adb(name + "-seed-temporary-cleanup", "shell", "rm", temporary)
+        assert database(name + "-native-seed") == expected
+        return expected
+
+    def assert_pending(value, case, dispatched):
+        assert len(value["pending_mutations"]) == 1
+        row = value["pending_mutations"][0]
+        assert row == {"sequence": 1, "kind": case["kind"], "item_id": "death-001", "payload": row["payload"],
+                       "client_mutation_id": case["clientMutationId"], "payload_hash": case["payloadHash"],
+                       "terminal_error": None, "dispatched": int(dispatched)}
+        assert json.loads(row["payload"]) == case["payload"]
+        assert value["sync_metadata"] == [{"singleton": 1, "last_successful_refresh_at": None, "last_acknowledgement": None}]
+        assert value["mutation_conflicts"] == []
+        assert value["local_identity"] == [{"singleton": 1, "next_id": 2 if case["case"] == "A" else 1}]
+        assert value["sqlite_sequence"] == [{"name": "pending_mutations", "seq": 1}]
+        assert len(value["items"]) == 1
+        item = value["items"][0]
+        assert item == {"id": "death-001", "title": case["finalItem"]["title"], "completed": 0,
+                        "version": case["finalItem"]["version"], "updated_at": item["updated_at"]}
+        assert item["updated_at"] >= 1700000000000
+
+    def mutation_evidence(case, applied, duplicate):
+        value = remote("/__mutation-state")
+        assert (value["appliedCount"], value["duplicateCount"], value["conflictCount"], value["hashRejectedCount"]) == (applied, duplicate, 0, 0)
+        assert len(value["attempts"]) == applied + duplicate
+        for index, attempt in enumerate(value["attempts"]):
+            assert attempt["wireBody"] == case["wireBody"]
+            assert attempt["declaredHash"] == attempt["actualHash"] == case["payloadHash"]
+            assert attempt["canonical"] == case["canonical"]
+            assert attempt["status"] == case["status"] and attempt["response"] == {"item": case["finalItem"]}
+            assert attempt["outcome"] == ("applied" if index == 0 else "duplicate")
+        return value
+
+    fixture, original_network = None, None
+    started = time.monotonic()
+    try:
+        with socket.socket() as probe:
+            assert probe.connect_ex(("127.0.0.1", 18081)) != 0, "Fixture port18081 already owned"
+        with (evidence / "fixture.log").open("wb") as log:
+            result["fixtureCommand"] = [args.node, str(root / "fixture/server.cjs")]
+            fixture = subprocess.Popen(result["fixtureCommand"], stdout=log, stderr=subprocess.STDOUT)
+        result["fixturePid"] = fixture.pid
+        deadline = time.monotonic() + 5
+        while True:
+            assert fixture.poll() is None
+            try:
+                remote()
+                break
+            except Exception:
+                assert time.monotonic() < deadline
+                time.sleep(0.1)
+        original_network = network("initial")
+        result["networkBefore"] = original_network
+        assert original_network == ONLINE
+        assert "Success" in adb("install-app", "install", "-r", str(Path(args.apk).resolve()))
+        adb("wake", "shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("keyguard", "shell", "wm", "dismiss-keyguard")
+        adb("clear-logcat", "logcat", "-c")
+        for case in inputs["cases"]:
+            name = case["case"]
+            current = {"case": name, "status": "RUNNING"}
+            result["cases"].append(current)
+            adb(name + "-setup-stop", "shell", "am", "force-stop", PACKAGE)
+            assert adb(name + "-setup-clear", "shell", "pm", "clear", PACKAGE) == "Success"
+            quiet_reset(name + "-setup-quiet")
+            reset = remote("/__m09-reset", {"case": name})
+            assert reset == {"items": case["initialItems"], "nextTimestamp": inputs["nextTimestamp"], "requests": [], "delayMs": 0}
+            current["networkBeforeSeed"] = network(name + "-setup-network", OFFLINE if name == "A" else ONLINE)
+            seeded = seed(case)
+            current.update(seedSha256=sha(evidence / (name + "-setup.db")), setupLastCommandIndex=len(commands) - 1,
+                           productionMutationInvoked=not (args.baseline and name == "A"))
+            extras = () if args.baseline and name == "A" else ("--es", "m07MutationIdentity", case["clientMutationId"])
+            if not args.baseline and name == "A":
+                extras += ("--ez", "m09FixedIdentity", "true")
+            current["pidBefore"] = launch(name + "-first-launch", extras)
+            current["firstLaunchCommandIndex"] = current["setupLastCommandIndex"] + 1
+            find("Local storage ready")
+            assert database(name + "-opened") == seeded
+            current["mutationStartedAt"] = int(time.time() * 1000)
+            if name == "A" and not args.baseline:
+                tap("New item title")
+                adb("create-text", "shell", "input", "text", "Recovered%screate")
+                adb("create-back", "shell", "input", "keyevent", "KEYCODE_BACK")
+                tap("Add item")
+            elif name == "B":
+                tap("Edit Initial")
+                tap("Edit item title")
+                adb("rename-end", "shell", "input", "keyevent", "KEYCODE_MOVE_END")
+                adb("rename-clear", "shell", "input", "keyevent", *(["KEYCODE_DEL"] * len("Initial")))
+                adb("rename-text", "shell", "input", "text", "Recovered%supdate")
+                adb("rename-back", "shell", "input", "keyevent", "KEYCODE_BACK")
+                tap("Save title")
+            find("Local storage ready")
+            find("Pending uploads: 1")
+            find(case["finalItem"]["title"], "text")
+            current["mutationFinishedAt"] = int(time.time() * 1000)
+            snapshot(name + "-committed-local-ui")
+            before = database(name + "-before-send")
+            assert_pending(before, case, False)
+            assert before["remote_versions"] == seeded["remote_versions"]
+            assert remote()["requests"] == []
+            mutation_evidence(case, 0, 0)
+            if name == "B":
+                remote("/__m09-arm", {"clientMutationId": case["clientMutationId"]})
+                # The ONLY Sync tap in B establishes the pre-death in-flight work.
+                tap("Synchronize")
+                deadline = time.monotonic() + 10
+                while True:
+                    hold = remote("/__m09-hold")
+                    if hold["phase"] == "COMMITTED_HELD":
+                        break
+                    assert time.monotonic() < deadline, "Fixture never committed/held"
+                    time.sleep(0.05)
+                assert hold["headersSent"] is False and hold["maxHoldMs"] == inputs["maxHoldMs"]
+                assert hold["deadlineAt"] - hold["committedAt"] == inputs["maxHoldMs"]
+                assert hold["canonical"] == case["canonical"] and hold["payloadHash"] == case["payloadHash"]
+                assert hold["response"] == {"item": case["finalItem"]} and hold["status"] == 200
+                current["committedHeld"] = hold
+                assert remote()["items"] == [case["finalItem"]]
+                current["committedMutationEvidence"] = mutation_evidence(case, 1, 0)
+                expected = json.loads(json.dumps(before))
+                expected["pending_mutations"][0]["dispatched"] = 1
+                before = database(name + "-held-before-kill", hold["deadlineAt"])
+                assert before == expected
+            else:
+                assert network(name + "-before-kill-network") == OFFLINE
+            assert fixture.poll() is None and os.getpid() == result["hostPid"]
+            assert adb(name + "-live-before-kill", "shell", "pidof", PACKAGE) == current["pidBefore"]
+            if name == "B":
+                boundary_hold = remote("/__m09-hold")
+                current["heldImmediatelyBeforeKill"] = boundary_hold
+                assert boundary_hold["phase"] == "COMMITTED_HELD"
+                assert boundary_hold["headersSent"] is False and boundary_hold["connectionClosed"] is False
+                for key in ("committedAt", "deadlineAt", "canonical", "payloadHash"):
+                    assert boundary_hold[key] == hold[key]
+            current["killCommandIndex"] = len(commands)
+            kill_timeout = 10 if name == "A" else min(10, (hold["deadlineAt"] - int(time.time() * 1000)) / 1000)
+            assert kill_timeout > 0
+            adb(name + "-kill", "shell", "am", "force-stop", PACKAGE, timeout=kill_timeout)
+            current["pidAbsent"] = adb(name + "-absent", "shell", "pidof", PACKAGE, check=False, timeout=5)
+            assert commands[-1]["exit"] in (0, 1) and not current["pidAbsent"]
+            current["deathConfirmedAt"] = int(time.time() * 1000)
+            if name == "B":
+                dropped = remote("/__m09-drop", {})
+                assert dropped["phase"] == "DROPPED" and current["deathConfirmedAt"] <= dropped["endedAt"] < hold["deadlineAt"]
+                current["heldResponseDrop"] = dropped
+            assert database(name + "-after-kill") == before
+            assert fixture.poll() is None and os.getpid() == result["hostPid"]
+            current["fixturePidAcrossDeath"] = fixture.pid
+            current["mutationsBeforeColdLaunch"] = mutation_evidence(case, 1 if name == "B" else 0, 0)
+            current["remoteBeforeColdLaunch"] = remote()
+            assert current["remoteBeforeColdLaunch"]["items"] == ([case["finalItem"]] if name == "B" else [])
+            assert len(current["remoteBeforeColdLaunch"]["requests"]) == (1 if name == "B" else 0)
+            quiet_reset(name + "-death-quiet")
+            if name == "A":
+                network(name + "-reconnect", ONLINE)
+            current["coldLaunchCommandIndex"] = len(commands)
+            # Ordinary Activity launch: no test extras, saved intent or Sync event.
+            current["pidAfter"] = launch(name + "-cold-launch")
+            assert current["pidAfter"] != current["pidBefore"]
+            recovered, observations = False, []
+            deadline = time.monotonic() + inputs["startupTimeoutSeconds"]
+            while time.monotonic() < deadline:
+                labels = {node.get("content-desc") for node in snapshot().iter("node")}
+                observations.append({"at": int(time.time() * 1000), "status": sorted(label for label in labels if label and
+                                     (label.startswith("Sync status:") or label.startswith("Pending uploads:") or label.startswith("Local storage ")))})
+                if {"Sync status: fresh", "Pending uploads: 0", "Local storage ready"}.issubset(labels):
+                    recovered = True
+                    break
+            current["startupObservations"] = observations
+            final_ui = snapshot(name + "-cold-result-ui")
+            assert any(node.get("text") == case["finalItem"]["title"] for node in final_ui.iter("node"))
+            assert any(node.get("content-desc") == "Item count: 1" for node in final_ui.iter("node"))
+            after = database(name + "-cold-result")
+            current["remoteAfter"] = remote()
+            current["mutationsAfter"] = mutation_evidence(case, 1 if recovered or name == "B" else 0, 1 if recovered and name == "B" else 0)
+            if not recovered:
+                assert args.baseline, "Startup did not drain durable intent without a Sync tap"
+                assert {"Sync status: stale", "Pending uploads: 1", "Local storage ready"}.issubset(labels)
+                assert after == before
+                assert current["remoteAfter"]["items"] == ([case["finalItem"]] if name == "B" else [])
+                assert len(current["remoteAfter"]["requests"]) == (1 if name == "B" else 0)
+                current["status"] = "BASELINE_STARTUP_LIMIT_REPRODUCED"
+            else:
+                final = case["finalItem"]
+                assert after["items"] == [{"id": final["id"], "title": final["title"], "completed": 0,
+                                           "version": final["version"], "updated_at": final["updatedAt"]}]
+                assert after["pending_mutations"] == after["mutation_conflicts"] == []
+                metadata = after["sync_metadata"][0]
+                assert json.loads(metadata["last_acknowledgement"]) == case["expectedAcknowledgement"]
+                assert metadata["last_successful_refresh_at"] > 0
+                assert after["local_identity"] == before["local_identity"] and after["sqlite_sequence"] == before["sqlite_sequence"]
+                known = after["remote_versions"]
+                assert len(known) == 1 and json.loads(known[0]["canonical_item"]) == final
+                assert current["remoteAfter"]["items"] == [final] and current["remoteAfter"]["nextTimestamp"] == 1700000601000
+                assert [(event["method"], event["path"]) for event in current["remoteAfter"]["requests"]] == \
+                    [(case["method"], case["path"])] * (2 if name == "B" else 1) + [("GET", "/items")]
+                current["status"] = "PASS_BASELINE_ALREADY_CORRECT" if args.baseline else "PASS"
+            assert fixture.poll() is None
+            save()
+        result["status"] = "BASELINE_LIMIT_REPRODUCED" if any(case["status"] == "BASELINE_STARTUP_LIMIT_REPRODUCED" for case in result["cases"]) else "PASS"
+    except Exception as error:
+        result.update(status="FAIL", error=repr(error))
+    finally:
+        try:
+            adb("logcat", "logcat", "-d", "-v", "threadtime")
+            adb("cleanup-stop", "shell", "am", "force-stop", PACKAGE)
+            result["pidAfterCleanup"] = adb("cleanup-pid", "shell", "pidof", PACKAGE, check=False)
+            assert not result["pidAfterCleanup"]
+            result["networkAfter"] = network("cleanup-network", original_network or ONLINE)
+            assert result["networkAfter"] == (original_network or ONLINE)
+        except Exception as error:
+            result.update(status="FAIL", cleanupError=repr(error))
+        if fixture is not None:
+            try:
+                if fixture.poll() is None and remote("/__m09-hold")["phase"] == "COMMITTED_HELD":
+                    result["cleanupHeldDrop"] = remote("/__m09-drop", {})
+                fixture.terminate()
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                assert result["fixtureExit"] == 0
+            except Exception as error:
+                fixture.kill()
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                result.update(status="FAIL", fixtureCleanupError=repr(error))
+        result.update(elapsedSeconds=time.monotonic() - started, adbCommands=len(commands))
+        save()
+        print(json.dumps(result), flush=True)
+    return 1 if result["status"] == "FAIL" else 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/verification/M09-inputs.json b/verification/M09-inputs.json
new file mode 100644
index 0000000..514b073
--- /dev/null
+++ b/verification/M09-inputs.json
@@ -0,0 +1,179 @@
+{
+  "profile": "phase-1",
+  "thread": "M09",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "f999421754533b3f112ad5b9a6aacac54b4b6b05",
+  "schemaVersion": 5,
+  "schemaSql": [
+    "CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL)",
+    "CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0))",
+    "INSERT INTO local_identity VALUES (1, 1)",
+    "CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), last_successful_refresh_at INTEGER CHECK(last_successful_refresh_at IS NULL OR last_successful_refresh_at >= 0), last_acknowledgement TEXT)",
+    "INSERT INTO sync_metadata VALUES (1, NULL, NULL)",
+    "CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL CHECK(kind IN ('create', 'rename', 'toggle', 'delete')), item_id TEXT NOT NULL, payload TEXT CHECK(payload IS NOT NULL OR kind = 'delete'), client_mutation_id TEXT NOT NULL, payload_hash TEXT NOT NULL, terminal_error TEXT CHECK(terminal_error IS NULL OR terminal_error = 'identity_conflict'), dispatched INTEGER NOT NULL CHECK(dispatched IN (0, 1)))",
+    "CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id)",
+    "INSERT INTO sqlite_sequence (name, seq) VALUES ('pending_mutations', 0)",
+    "CREATE TABLE remote_versions (id TEXT PRIMARY KEY NOT NULL, version INTEGER NOT NULL CHECK(version > 0), updated_at INTEGER, deleted INTEGER NOT NULL CHECK(deleted IN (0, 1)), canonical_item TEXT CHECK((deleted = 0 AND canonical_item IS NOT NULL) OR (deleted = 1 AND canonical_item IS NULL)))",
+    "CREATE TABLE mutation_conflicts (client_mutation_id TEXT PRIMARY KEY NOT NULL, intent TEXT NOT NULL, reason TEXT NOT NULL CHECK(reason IN ('version_conflict', 'unversioned_legacy')), item TEXT, tombstone TEXT)",
+    "PRAGMA user_version = 5"
+  ],
+  "fixturePort": 18081,
+  "nextTimestamp": 1700000600000,
+  "maxHoldMs": 30000,
+  "startupTimeoutSeconds": 20,
+  "uiTimeoutSeconds": 20,
+  "resetTimeoutSeconds": 10,
+  "resetQuietSeconds": 1,
+  "baselineLocalTimestamp": 1700000599000,
+  "baselineSetup": "A uses an approved schema5 fixed-input seed under proven device-offline control BEFORE first unchanged-M08 launch; production create is not invoked. Fixed A creates via production UI. B always renames via production UI. No state injection after initial launch or across death.",
+  "cases": [
+    {
+      "case": "A",
+      "kind": "create",
+      "clientMutationId": "m09-create-001",
+      "method": "POST",
+      "path": "/items",
+      "payload": {
+        "id": "death-001",
+        "title": "Recovered create",
+        "completed": false
+      },
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"death-001\",\"title\":\"Recovered create\"}}",
+      "payloadHash": "4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4",
+      "wireBody": {
+        "id": "death-001",
+        "title": "Recovered create",
+        "completed": false,
+        "clientMutationId": "m09-create-001",
+        "payloadHash": "4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4"
+      },
+      "status": 201,
+      "initialItems": [],
+      "seedStatements": [],
+      "baselineOnlyStatements": [
+        {
+          "sql": "INSERT INTO items (id,title,completed,version,updated_at) VALUES (?,?,?,?,?)",
+          "params": [
+            "death-001",
+            "Recovered create",
+            0,
+            1,
+            1700000599000
+          ]
+        },
+        {
+          "sql": "INSERT INTO pending_mutations (kind,item_id,payload,client_mutation_id,payload_hash,terminal_error,dispatched) VALUES (?,?,?,?,?,NULL,0)",
+          "params": [
+            "create",
+            "death-001",
+            "{\"completed\":false,\"id\":\"death-001\",\"title\":\"Recovered create\"}",
+            "m09-create-001",
+            "4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4"
+          ]
+        },
+        {
+          "sql": "UPDATE local_identity SET next_id=2 WHERE singleton=1",
+          "params": []
+        }
+      ],
+      "finalItem": {
+        "id": "death-001",
+        "title": "Recovered create",
+        "completed": false,
+        "version": 1,
+        "updatedAt": 1700000600000
+      },
+      "expectedAcknowledgement": {
+        "clientMutationId": "m09-create-001",
+        "payloadHash": "4af45629e87641c772fbdd173149b3b1a8779ef74699af2b1768d2599a0a3bd4",
+        "status": 201,
+        "result": {
+          "item": {
+            "id": "death-001",
+            "title": "Recovered create",
+            "completed": false,
+            "version": 1,
+            "updatedAt": 1700000600000
+          }
+        }
+      },
+      "expectedAppliedCount": 1,
+      "expectedDuplicateCount": 0
+    },
+    {
+      "case": "B",
+      "kind": "rename",
+      "clientMutationId": "m09-update-001",
+      "method": "PATCH",
+      "path": "/items/death-001",
+      "payload": {
+        "title": "Recovered update",
+        "baseVersion": 1
+      },
+      "canonical": "{\"method\":\"PATCH\",\"path\":\"/items/death-001\",\"payload\":{\"baseVersion\":1,\"title\":\"Recovered update\"}}",
+      "payloadHash": "53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12",
+      "wireBody": {
+        "title": "Recovered update",
+        "baseVersion": 1,
+        "clientMutationId": "m09-update-001",
+        "payloadHash": "53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12"
+      },
+      "status": 200,
+      "initialItems": [
+        {
+          "id": "death-001",
+          "title": "Initial",
+          "completed": false,
+          "version": 1,
+          "updatedAt": 1700000000000
+        }
+      ],
+      "seedStatements": [
+        {
+          "sql": "INSERT INTO items (id,title,completed,version,updated_at) VALUES (?,?,?,?,?)",
+          "params": [
+            "death-001",
+            "Initial",
+            0,
+            1,
+            1700000000000
+          ]
+        },
+        {
+          "sql": "INSERT INTO remote_versions (id,version,updated_at,deleted,canonical_item) VALUES (?,?,?,?,?)",
+          "params": [
+            "death-001",
+            1,
+            1700000000000,
+            0,
+            "{\"completed\":false,\"id\":\"death-001\",\"title\":\"Initial\",\"updatedAt\":1700000000000,\"version\":1}"
+          ]
+        }
+      ],
+      "baselineOnlyStatements": [],
+      "finalItem": {
+        "id": "death-001",
+        "title": "Recovered update",
+        "completed": false,
+        "version": 2,
+        "updatedAt": 1700000600000
+      },
+      "expectedAcknowledgement": {
+        "clientMutationId": "m09-update-001",
+        "payloadHash": "53cba8ff4fb268be35bb6c26b1a61903143e7237b78f3065996c7469c7ccda12",
+        "status": 200,
+        "result": {
+          "item": {
+            "id": "death-001",
+            "title": "Recovered update",
+            "completed": false,
+            "version": 2,
+            "updatedAt": 1700000600000
+          }
+        }
+      },
+      "expectedAppliedCount": 1,
+      "expectedDuplicateCount": 1
+    }
+  ]
+}
diff --git a/verification/M09.md b/verification/M09.md
new file mode 100644
index 0000000..3b6f446
--- /dev/null
+++ b/verification/M09.md
@@ -0,0 +1,27 @@
+# M09 verification — phase-1
+
+- Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: verified/tagged M08 `f999421754533b3f112ad5b9a6aacac54b4b6b05`.
+- Attempt1; repairs0/2. Current status: **unchanged-app baseline accepted by main**; implementation and final verification pending.
+- Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/`.
+
+## Frozen baseline
+
+[Baseline manifest v2](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-manifest-v2.json) records exact `baselineArgv`, working directory, source/input/support hashes and preserved M08 APK `4dc8e86b32b1ca9f72fdf578a57fc197f7c1e81416725111c03e30278b98bbc6`. Run that argv only with the exclusive device lease and a fresh evidence destination. The original preflight manifest/copies remain preserved; before any device execution, main requested seven additional harness lines confirming an open, headerless held response immediately before B's kill. No input, timeout, barrier or expected outcome changed. [Inputs](/private/tmp/mobile-systems-evolution-ed7baa2/react-native/verification/M09-inputs.json) freeze all SQL, identities, payloads, canonical hashes and expected results.
+
+Baseline A uses main-approved fixed input: with the actual device offline and app absent, copy the exact schema5 database into private storage before the first unchanged-M08 launch. **Production create was not invoked for this baseline input.** B performs the rename through the real UI after its initial canonical seed. No database injection, reinstall, clear or Sync tap occurs across either tested death/cold-launch boundary. Fixed A must use production UI creation.
+
+Support checks passed: typecheck1.356s, real HTTP sync suite19/19 in2.129s including the committed-response hold/drop/replay control, and independent canonical-hash/schema/seed checks. The seven-line preflight addition passed syntax and exact-byte comparison; the host suite was not rerun for that observation-only change. Exact commands, outputs and snapshots remain under the evidence root.
+
+The single [baseline invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-android-01.command.json) exited0 in107.070s (harness106.845s,239 adb commands): **BASELINE_LIMIT_REPRODUCED**, not final acceptance PASS. Eleven raw native database archives/SQLite projections preserve both cases:
+
+- A: PID19660→absent→20201. The exact create identity/hash remained pending, dispatched0/ACKnull; remote empty, applied0/duplicate0 after ordinary online cold launch.
+- B: PID20700→absent→21073. Server committed v2/t1700000600000 at1787916322267; the fresh response observation remained open with no headers. Death was confirmed1787916322644; explicit drop followed1787916322645, within the unchanged30000ms maximum. The original base1 identity/hash remained pending, dispatched1/ACKnull; applied1/duplicate0 after ordinary cold launch.
+
+[Raw result and adjacent commands/HTTP/DB/UI evidence](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-android-01/result.json), [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-baseline-audit.json) and [cleanup readback](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-cleanup-readback.json) retain the complete proof. Host32023 and fixture32034 survived both app deaths; fixture exited0 and was directly absent, port18081 free, app absent, original network0/1/1 active restored. All59 execution files and the APK remained exact. The baseline-only lease was released; no repeat occurred.
+
+## Implementation and final verification plan
+
+Main authorized only startup recovery through the existing foreground sync and the debug-only deterministic Item prefix needed for fixed A. No schema, hashing, acknowledgment, conflict algorithm, scheduler or new dependency is planned. Final device execution remains main-owned on a frozen candidate: M09 A/B, existing M08 actual Activity recreation and native CRUD.
+
+Historical M05/M06/M07 external harnesses remain unchanged. Their explicit-first-Sync timing predates M09: offline startup now may mark the head dispatched before a failed fetch; online startup may complete a replay before a manual tap. Current host regressions must cover that interaction without disabling startup recovery. Prior independently verified core Android evidence is reused only for unchanged storage/sync/protocol behavior; it is not reported as a new M09 execution.


