# M10 — Persistent Background Sync와 Network Constraints

## `test(m10): preserve OS-work prerequisite baseline`

diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 2b9f645..b8cd6e8 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -135,6 +135,34 @@ async function control(path: string, body: unknown) {
   return response.json();
 }
 
+test.each(['A', 'B'] as const)('M10 fixture %s freezes three HTTP outcomes and unchanged mutation identities', async scenario => {
+  const input = require('../verification/M10-inputs.json');
+  expect(mutationHash(input.method, input.path, input.payload)).toBe(input.payloadHash);
+  expect(await control('/__m10-reset', {case: scenario})).toEqual({
+    items: [], nextTimestamp: 1700000700000, requests: [], case: scenario, httpAttempts: 0,
+  });
+  for (const status of input.expectedStatuses[scenario]) {
+    const response = await request(url + input.path, {method: input.method,
+      headers: {'Content-Type': 'application/json'}, body: JSON.stringify(input.wireBody)});
+    expect(response.status).toBe(status);
+    expect(await response.json()).toEqual(status === 201 ? {item: input.finalItem} : {error: 'Temporary M10 failure'});
+  }
+  const state = fixture.state();
+  expect(state.requests).toHaveLength(3);
+  expect(state.requests.map(event => event.status)).toEqual(input.expectedStatuses[scenario]);
+  expect(state.items).toEqual(scenario === 'A' ? [input.finalItem] : []);
+  expect(state.nextTimestamp).toBe(scenario === 'A' ? 1700000701000 : 1700000700000);
+  expect(fixture.mutationState()).toMatchObject({appliedCount: scenario === 'A' ? 1 : 0,
+    duplicateCount: 0, conflictCount: 0, hashRejectedCount: 0});
+  expect(fixture.mutationState().attempts).toEqual(input.expectedStatuses[scenario].map((status: number) =>
+    expect.objectContaining({status, wireBody: input.wireBody, clientMutationId: input.clientMutationId,
+      canonical: input.canonical, declaredHash: input.payloadHash, actualHash: input.payloadHash,
+      outcome: status === 201 ? 'applied' : 'temporary_failure'})));
+  const observed = await (await request(url + '/__m10-state', {method: 'GET', headers: {}})).json();
+  expect(observed).toEqual({...state, case: scenario, httpAttempts: 3});
+  console.info('M10_FIXTURE_ONLY_HTTP', JSON.stringify({scenario, state: observed, mutations: fixture.mutationState()}));
+});
+
 test('M09 fixture commits before holding ACK, serves controls, then replays the original receipt after drop', async () => {
   const inputs = require('../verification/M09-inputs.json');
   const scenario = inputs.cases[1];
diff --git a/fixture/server.cjs b/fixture/server.cjs
index f74d062..ec9f9e2 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03–M09 fixture, not a backend service.
+// A disposable deterministic M03–M10 fixture, not a backend service.
 const http = require('node:http');
 const {createHash} = require('node:crypto');
 
@@ -26,6 +26,8 @@ function createFixture() {
   let dropFirstCreate;
   let dropMutationId;
   let tombstones;
+  let m10Case = null;
+  let m10HttpAttempts = 0;
   let heldResponse = null;
   let holdTimer = null;
   let hold = {phase: 'IDLE', maxHoldMs: 30000};
@@ -54,6 +56,8 @@ function createFixture() {
     dropFirstCreate = false;
     dropMutationId = null;
     tombstones = new Map();
+    m10Case = null;
+    m10HttpAttempts = 0;
   }
   reset();
   const deletionState = () => tombstones.size ? {tombstones: deletedSnapshot()} : {};
@@ -66,8 +70,11 @@ function createFixture() {
     let mutation;
     let clientMutationId;
     const {method, url: path} = request;
+    const m10Attempt = m10Case !== null && !path.startsWith('/__') ? ++m10HttpAttempts : null;
+    const receivedAt = m10Attempt === null ? null : Date.now();
     function reply(status, payload, outcome = 'rejected') {
-      if (!path.startsWith('/__')) {requests.push({method, path, body: body ?? null, status, response: payload});}
+      if (!path.startsWith('/__')) {requests.push({method, path, body: body ?? null, status, response: payload,
+        ...(m10Attempt === null ? {} : {m10Attempt, receivedAt})});}
       const applied = mutation && status >= 200 && status < 300 && outcome !== 'duplicate';
       if (applied) {
         mutationEvidence.appliedCount += 1;
@@ -114,6 +121,17 @@ function createFixture() {
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
       if (method === 'GET' && path === '/__mutation-state') {return reply(200, mutationEvidence);}
+      if (method === 'POST' && path === '/__m10-reset') {
+        if (body?.case !== 'A' && body?.case !== 'B') {return reply(400, {error: 'M10 case A or B required'});}
+        reset();
+        items.clear();
+        nextTimestamp = 1700000700000;
+        m10Case = body.case;
+        return reply(200, {...state(), case: m10Case, httpAttempts: m10HttpAttempts});
+      }
+      if (method === 'GET' && path === '/__m10-state') {
+        return reply(200, {...state(), case: m10Case, httpAttempts: m10HttpAttempts});
+      }
       if (method === 'POST' && path === '/__m09-reset') {
         if (body?.case !== 'A' && body?.case !== 'B') {return reply(400, {error: 'M09 case A or B required'});}
         reset();
@@ -225,6 +243,9 @@ function createFixture() {
         }
       }
       if (method === 'POST' && path === '/items') {
+        if (m10Case !== null && (m10Case === 'B' || m10Attempt <= 2)) {
+          return reply(503, {error: 'Temporary M10 failure'}, 'temporary_failure');
+        }
         if (!body || typeof body.id !== 'string' || !/^[a-zA-Z0-9_-]+$/.test(body.id)
             || typeof body.title !== 'string' || !body.title.trim() || typeof body.completed !== 'boolean') {
           return reply(400, {error: 'id, title, and completed are required'});
diff --git a/scripts/verify_m10_baseline.py b/scripts/verify_m10_baseline.py
new file mode 100644
index 0000000..a799ebf
--- /dev/null
+++ b/scripts/verify_m10_baseline.py
@@ -0,0 +1,395 @@
+#!/usr/bin/env python3
+"""M10 unchanged-M09 prerequisite baseline; exclusive approved lease required.
+
+This charges case A but cannot pass A: the unchanged APK has no OS work.
+The exact input is seeded before first launch; only read-only observations follow
+the verified same-UID background process kill, apart from connectivity controls.
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
+    parser.add_argument("--baseline-prerequisite", action="store_true", required=True)
+    args = parser.parse_args()
+    args.baseline = True
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs_path = root / "verification/M10-inputs.json"
+    inputs = json.loads(inputs_path.read_text())
+    sha = lambda path: hashlib.sha256(Path(path).read_bytes()).hexdigest()
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "hostPid": os.getpid(), "serial": args.serial,
+              "apkSha256": sha(args.apk), "inputsSha256": sha(inputs_path), "harnessSha256": sha(__file__),
+              "fixtureSha256": sha(root / "fixture/server.cjs"),
+              "resetPredicateSourceSha256": sha(root / "scripts/verify_m07.py"), "caseBudget": "Root-owned verification-budget.json; see immutable invocation manifest", "acceptance": "PREREQUISITE_LIMIT_ONLY"}
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
+        adb("ui-dump", "shell", "uiautomator", "dump", "/sdcard/mse-m10-baseline-ui.xml")
+        xml = adb("ui-read", "exec-out", "cat", "/sdcard/mse-m10-baseline-ui.xml")
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
+        temporary = "/data/local/tmp/mse-m10-seed.db"
+        adb(name + "-seed-push", "push", str(path), temporary)
+        adb(name + "-seed-directory", "shell", "run-as", PACKAGE, "mkdir", "-p", "files")
+        adb(name + "-seed-copy", "shell", "run-as", PACKAGE, "cp", temporary, "files/items.db")
+        adb(name + "-seed-temporary-cleanup", "shell", "rm", temporary)
+        assert database(name + "-native-seed") == expected
+        return expected
+
+
+    def no_process(label):
+        value = adb(label, "shell", "pidof", PACKAGE, check=False)
+        assert commands[-1]["exit"] in (0, 1) and value == "", value
+        return {"pid": value, "commandIndex": len(commands) - 1, "at": int(time.time() * 1000)}
+
+    def scheduler_absent(label):
+        dump = adb(label + "-jobs", "shell", "dumpsys", "jobscheduler", PACKAGE)
+        registered = [line.strip() for line in dump.splitlines()
+                      if re.match(r"\s*JOB #", line) and PACKAGE in line]
+        files = adb(label + "-files", "shell", "run-as", PACKAGE, "find", ".", "-maxdepth", "3", "-type", "f")
+        assert not registered and "androidx.work.workdb" not in files, (registered, files)
+        return {"registeredJobs": registered, "workDatabasePresent": False,
+                "jobDumpCommandIndex": len(commands) - 2, "fileListCommandIndex": len(commands) - 1}
+
+    def not_stopped(label):
+        dump = adb(label, "shell", "dumpsys", "package", PACKAGE)
+        state = re.search(r"(?m)^\s*User 0:[^\n]*", dump)
+        assert state and re.search(r"\bstopped=false\b", state.group()), dump
+        return state.group().strip()
+
+    def assert_no_http():
+        value = remote("/__m10-state")
+        assert value == {"items": [], "nextTimestamp": inputs["nextTimestamp"], "requests": [],
+                         "case": "A", "httpAttempts": 0}, value
+        mutation = remote("/__mutation-state")
+        assert mutation == {"appliedCount": 0, "duplicateCount": 0, "conflictCount": 0,
+                            "hashRejectedCount": 0, "attempts": []}, mutation
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
+        assert "Success" in adb("install-unchanged-m09", "install", "-r", str(Path(args.apk).resolve()))
+        adb("wake", "shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("keyguard", "shell", "wm", "dismiss-keyguard")
+        adb("clear-logcat", "logcat", "-c")
+        adb("setup-stop", "shell", "am", "force-stop", PACKAGE)
+        assert adb("setup-clear", "shell", "pm", "clear", PACKAGE) == "Success"
+        quiet_reset("setup-quiet")
+        assert remote("/__m10-reset", {"case": "A"}) == {
+            "items": [], "nextTimestamp": inputs["nextTimestamp"], "requests": [], "case": "A", "httpAttempts": 0}
+        result["networkBeforeSeed"] = network("setup-offline", OFFLINE)
+        seeded = seed({"case": "baseline", "seedStatements": inputs["baselineSeedStatements"], "baselineOnlyStatements": []})
+        result["productionMutationInvoked"] = False
+        result["setupLastCommandIndex"] = len(commands) - 1
+        result["pidBefore"] = launch("ordinary-m09-launch")
+        result["firstLaunchCommandIndex"] = result["setupLastCommandIndex"] + 1
+        find("Local storage ready")
+        find("Pending uploads: 1")
+        find("Sync status: error")
+        find("Could not refresh: Network request failed", "text")
+        find("Background item", "text")
+        snapshot("committed-offline-ui")
+        opened = database("committed-offline")
+        expected = json.loads(json.dumps(seeded))
+        expected["pending_mutations"][0]["dispatched"] = 1
+        assert opened == expected, (opened, expected)
+        assert len(opened["pending_mutations"]) == 1
+        row = opened["pending_mutations"][0]
+        assert row["client_mutation_id"] == inputs["clientMutationId"] and row["payload_hash"] == inputs["payloadHash"]
+        assert json.loads(row["payload"]) == inputs["payload"]
+        assert_no_http()
+        result["schedulerBeforeLoss"] = scheduler_absent("before-loss")
+        adb("background-home", "shell", "input", "keyevent", "KEYCODE_HOME")
+        deadline = time.monotonic() + 10
+        while True:
+            activities = adb("background-activities", "shell", "dumpsys", "activity", "activities")
+            resumed = [line.strip() for line in activities.splitlines() if "mResumedActivity:" in line or "topResumedActivity=" in line]
+            if resumed and all(PACKAGE not in line for line in resumed):
+                break
+            assert time.monotonic() < deadline, "App did not leave resumed Activity"
+            time.sleep(0.1)
+        result["backgroundResumedActivities"] = resumed
+        assert adb("verified-background-pid", "shell", "pidof", PACKAGE) == result["pidBefore"]
+        uid = adb("same-uid", "shell", "run-as", PACKAGE, "id", "-u")
+        process = adb("verified-process-uid", "shell", "ps", "-p", result["pidBefore"], "-o", "UID,PID,NAME")
+        records = process.splitlines()
+        assert len(records) == 2 and records[1].split() == [uid, result["pidBefore"], PACKAGE], process
+        result["sameUid"] = uid
+        result["packageBeforeLoss"] = not_stopped("package-before-loss")
+        assert network("before-loss-network") == OFFLINE
+        assert fixture.poll() is None and os.getpid() == result["hostPid"]
+        assert adb("immediate-pid-before-kill", "shell", "pidof", PACKAGE) == result["pidBefore"]
+        result["killCommandIndex"] = len(commands)
+        adb("same-uid-kill9", "shell", "run-as", PACKAGE, "kill", "-9", result["pidBefore"])
+        result["processLoss"] = no_process("absent-after-signal")
+        result["packageAfterLoss"] = not_stopped("package-after-loss")
+        result["schedulerAfterLoss"] = scheduler_absent("after-loss")
+        adb("exit-history-after-loss", "shell", "dumpsys", "activity", "exit-info", PACKAGE)
+        assert database("after-loss") == opened
+        assert_no_http()
+        result["networkAfterLoss"] = network("online-without-app-entry", ONLINE)
+        result["onlineFirstObservation"] = no_process("online-initially-absent")
+        result["observationStartedAt"] = int(time.time() * 1000)
+        # One fixed bounded observation, not a scheduler invocation or a backoff wait.
+        time.sleep(inputs["baselineObservationSeconds"])
+        result["onlineFinalObservation"] = no_process("online-finally-absent")
+        result["observationEndedAt"] = int(time.time() * 1000)
+        result["schedulerFinal"] = scheduler_absent("online-final")
+        result["packageFinal"] = not_stopped("package-online-final")
+        assert database("online-final") == opened
+        result["remoteFinal"] = assert_no_http()
+        assert fixture.poll() is None and os.getpid() == result["hostPid"]
+        result["fixturePidAcrossLoss"] = fixture.pid
+        result["boundaryLastCommandIndex"] = len(commands) - 1
+        result.update(status="LIMITATION_REPRODUCED", missingPrerequisite="No OS work is registered by unchanged M09",
+                      claimedOsDispatch=False, claimedFixedCasePass=False, pendingAfter=1, actualHttpAttempts=0)
+    except Exception as error:
+        result.update(status="FAIL", error=repr(error))
+    finally:
+        try:
+            adb("logcat", "logcat", "-d", "-v", "threadtime")
+            adb("cleanup-stop", "shell", "am", "force-stop", PACKAGE)
+            result["pidAfterCleanup"] = no_process("cleanup-pid")
+            result["networkAfter"] = network("cleanup-network", original_network or ONLINE)
+            assert result["networkAfter"] == (original_network or ONLINE)
+        except Exception as error:
+            result.update(status="FAIL", cleanupError=repr(error))
+        if fixture is not None:
+            try:
+                fixture.terminate()
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                assert result["fixtureExit"] == 0
+                probe = subprocess.run(["ps", "-p", str(fixture.pid), "-o", "pid="], capture_output=True, text=True)
+                result["fixtureProcessAfterCleanup"] = {"argv": ["ps", "-p", str(fixture.pid), "-o", "pid="],
+                                                        "exit": probe.returncode, "stdout": probe.stdout, "stderr": probe.stderr}
+                assert probe.returncode == 1 and not probe.stdout.strip()
+                with socket.socket() as port:
+                    result["fixturePortFree"] = port.connect_ex(("127.0.0.1", 18081)) != 0
+                assert result["fixturePortFree"]
+            except Exception as error:
+                if fixture.poll() is None:
+                    fixture.kill()
+                    result["fixtureExit"] = fixture.wait(timeout=5)
+                result.update(status="FAIL", fixtureCleanupError=repr(error))
+        result.update(elapsedSeconds=time.monotonic() - started, adbCommands=len(commands))
+        save()
+        print(json.dumps(result), flush=True)
+    return 1 if result["status"] == "FAIL" else 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/verification/M10-inputs.json b/verification/M10-inputs.json
new file mode 100644
index 0000000..0cee30f
--- /dev/null
+++ b/verification/M10-inputs.json
@@ -0,0 +1,104 @@
+{
+  "profile": "phase-1",
+  "thread": "M10",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "ed35bc3b686f52379b7cd16728171191a1386785",
+  "fixturePort": 18081,
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
+  "clientMutationId": "m10-create-001",
+  "method": "POST",
+  "path": "/items",
+  "payload": {
+    "id": "work-001",
+    "title": "Background item",
+    "completed": false
+  },
+  "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"work-001\",\"title\":\"Background item\"}}",
+  "payloadHash": "b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359",
+  "wireBody": {
+    "id": "work-001",
+    "title": "Background item",
+    "completed": false,
+    "clientMutationId": "m10-create-001",
+    "payloadHash": "b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359"
+  },
+  "initialItems": [],
+  "nextTimestamp": 1700000700000,
+  "finalItem": {
+    "id": "work-001",
+    "title": "Background item",
+    "completed": false,
+    "version": 1,
+    "updatedAt": 1700000700000
+  },
+  "retryCeiling": 3,
+  "backoffMs": [
+    10000,
+    20000
+  ],
+  "workerAttemptCounts": [
+    0,
+    1,
+    2
+  ],
+  "expectedStatuses": {
+    "A": [
+      503,
+      503,
+      201
+    ],
+    "B": [
+      503,
+      503,
+      503
+    ]
+  },
+  "baselineLocalTimestamp": 1700000699000,
+  "baselineObservationSeconds": 10,
+  "uiTimeoutSeconds": 20,
+  "resetTimeoutSeconds": 10,
+  "resetQuietSeconds": 1,
+  "networkTimeoutSeconds": 10,
+  "baselineSetup": "Exact disposable schema5 input seeded only while app absent and device actually offline, before ordinary unchanged-M09 launch. This is not production-create evidence. No Item/queue injection after launch or across death.",
+  "baselineSeedStatements": [
+    {
+      "sql": "INSERT INTO items (id,title,completed,version,updated_at) VALUES (?,?,?,?,?)",
+      "params": [
+        "work-001",
+        "Background item",
+        0,
+        1,
+        1700000699000
+      ]
+    },
+    {
+      "sql": "INSERT INTO pending_mutations (kind,item_id,payload,client_mutation_id,payload_hash,terminal_error,dispatched) VALUES (?,?,?,?,?,NULL,0)",
+      "params": [
+        "create",
+        "work-001",
+        "{\"completed\":false,\"id\":\"work-001\",\"title\":\"Background item\"}",
+        "m10-create-001",
+        "b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359"
+      ]
+    },
+    {
+      "sql": "UPDATE local_identity SET next_id=2 WHERE singleton=1",
+      "params": []
+    }
+  ],
+  "baselineResult": "LIMITATION_REPRODUCED: no registered OS work/new PID/HTTP after actual loss; not M10-A acceptance PASS",
+  "baselineBudget": "Root charges M10-A invocation1/3 at start; no B invocation."
+}
diff --git a/verification/M10.md b/verification/M10.md
new file mode 100644
index 0000000..5ee724a
--- /dev/null
+++ b/verification/M10.md
@@ -0,0 +1,19 @@
+# M10 verification — phase-1
+
+- Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: verified M09 `ed35bc3b686f52379b7cd16728171191a1386785`.
+- Status: **unchanged-M09 prerequisite limitation accepted by root**; product implementation and final A/B are NOT_RUN.
+- Attempt2; repair1/2 used. Fixed-case budget: A2/3, B0/3. Root owns remaining starts; no baseline rerun.
+- Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/`.
+
+## Frozen baseline and preserved failure
+
+The certified M09 APK remains `76c12efda957b0100198cbc7f735368ca1c86e04d53c5c0c48a1c750d3e7b511`; all60 original source files are preserved. Exact inputs, seed SQL, hashes and invocation argv are in the [accepted repair manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/repair1/manifest.json) (SHA256 `fb48e75e9277a2cb0471f807151c72d9706dd2eb9fec2d8ad5b51a01c7757b84`). The schema5 input is seeded only while absent/offline before ordinary launch; this is not production-create evidence.
+
+The [first baseline](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/baseline-android-01/result.json) failed before process loss (35.338s,93 adb commands): the harness incorrectly awaited `Sync status: stale`, while unchanged M09 correctly reported `error` and `Network request failed`. All195 raw files and the original frozen manifest remain preserved. Fresh repair1 corrected only that offline predicate and added a saved-XML regression:12 captures/60 assertions, independently rerun by root. Existing fixture/typecheck evidence remains unchanged (21 host tests PASS; typecheck PASS).
+
+## Accepted prerequisite result
+
+Root's [repaired invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/repair1-baseline-android-01/result.json) returned **LIMITATION_REPRODUCED** (33.819s,111 adb commands). Background PID26233 became absent after same-UID10238 SIGKILL; package remained `stopped=false`. Four native SQLite snapshots retain the exact work-001/m10-create-001 envelope: seed dispatched0, M09 offline startup dispatched1, then unchanged after loss and10s online. No app entrypoint, OS job, WorkDatabase or HTTP request followed loss. This does not claim OS dispatch or final A acceptance. [Root native/process audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-repair1-baseline-audit.json).
+
+Fixture73777 survived loss, exited0 and was directly absent; port18081 was free, app absent, and network0/1/1 with active129 restored. [Root cleanup audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-repair1-cleanup-audit.json). Both baselines were root-owned and charged to A; B has not executed. This commit contains only accepted baseline support and this ledger; product work awaits root's blob check.


