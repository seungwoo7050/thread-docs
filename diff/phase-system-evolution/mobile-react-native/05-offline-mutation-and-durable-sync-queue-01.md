# M05 — Offline Mutation과 Durable Sync Queue

## `test: reproduce lost upload intent after offline process death`

diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index dfb98a2..baf2d66 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -193,3 +193,46 @@ test('M04 a failed snapshot metadata write rolls back Items and the successful-r
   expect(await reopened.read()).toEqual(seeds);
   expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
 });
+
+test('M05 fixed four offline mutations survive reopening and drain as four accepted operations', async () => {
+  await control('/__mutation-clock', {nextTimestamp: 1700000300000});
+  let offline = false;
+  const transport: JsonRequest = (address, options) => offline
+    ? Promise.reject(new TypeError('Network request failed')) : request(address, options);
+  const name = 'm05-offline.db';
+  const store = await openItemStore(name);
+  const sync = new ForegroundSync(store, url, transport, 'device');
+  await sync.synchronize();
+  expect(await store.read()).toEqual(seeds);
+  offline = true;
+  await store.mutate({type: 'create', title: 'Gamma', now: 1700000300000}, sync.identityPrefix);
+  await store.mutate({type: 'rename', id: 'remote-001', title: 'Queued edit', now: 1700000301000});
+  await store.mutate({type: 'toggle', id: 'remote-001', now: 1700000302000});
+  await store.mutate({type: 'delete', id: 'remote-002'});
+  const local = await store.read();
+  expect(local).toEqual([
+    {...seeds[0], title: 'Queued edit', completed: true, version: 3, updatedAt: 1700000302000},
+    {...gamma, updatedAt: 1700000300000},
+  ]);
+  expect(fixture.state().requests).toHaveLength(1);
+  closeDatabases();
+  const reopened = await openItemStore(name);
+  expect(await reopened.read()).toEqual(local);
+  const restarted = new ForegroundSync(reopened, url, transport, 'device');
+  offline = false;
+  await restarted.synchronize();
+  const created = {...gamma, updatedAt: 1700000300000};
+  const edited = {...seeds[0], title: 'Queued edit', version: 2, updatedAt: 1700000301000};
+  const done = {...edited, completed: true, version: 3, updatedAt: 1700000302000};
+  const expected = [created, done];
+  console.info('M05_OFFLINE_RESTART', JSON.stringify({beforeRestart: local, afterDrain: await reopened.read(), remote: fixture.state()}));
+  expect(await reopened.read()).toEqual(expected);
+  expect(fixture.state()).toEqual({items: expected, nextTimestamp: 1700000304000, requests: [
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: seeds}},
+    {method: 'POST', path: '/items', body: {id: 'device-001', title: 'Gamma', completed: false}, status: 201, response: {item: created}},
+    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Queued edit'}, status: 200, response: {item: edited}},
+    {method: 'PATCH', path: '/items/remote-001', body: {completed: true}, status: 200, response: {item: done}},
+    {method: 'DELETE', path: '/items/remote-002', body: null, status: 200, response: {deletedId: 'remote-002'}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: expected}},
+  ]});
+});
diff --git a/fixture/server.cjs b/fixture/server.cjs
index 42785d4..6f95e6a 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03/M04 fixture, not a backend service.
+// A disposable deterministic M03/M04/M05 fixture, not a backend service.
 const http = require('node:http');
 
 const SEEDS = [
@@ -37,6 +37,13 @@ function createFixture() {
       if (raw) {body = JSON.parse(raw);}
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
+      if (method === 'POST' && path === '/__mutation-clock') {
+        if (!body || !Number.isSafeInteger(body.nextTimestamp) || body.nextTimestamp < 0) {
+          return reply(400, {error: 'Nonnegative mutation timestamp is required'});
+        }
+        nextTimestamp = body.nextTimestamp;
+        return reply(200, {nextTimestamp, delayMs: 0});
+      }
       // M04 controls affect only the explicitly selected GET or remote Item.
       // Every response still has zero artificial delay; there is no retry loop.
       if (method === 'POST' && path === '/__get-failures') {
diff --git a/scripts/verify_m05.py b/scripts/verify_m05.py
new file mode 100644
index 0000000..ccf8996
--- /dev/null
+++ b/scripts/verify_m05.py
@@ -0,0 +1,336 @@
+#!/usr/bin/env python3
+"""Frozen M05 Android offline mutations, real process death, and one drain.
+
+Requires an exclusive device lease. --baseline uses the unchanged M04 APK and
+records its missing upload intent; application behavior is never altered to fail.
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
+from urllib.request import Request, urlopen
+import xml.etree.ElementTree as ET
+
+PACKAGE = "com.mse.reactnative"
+URL = "http://127.0.0.1:18081"
+OFFLINE = {"airplane_mode_on": "1", "wifi_on": "0", "mobile_data": "0"}
+SEEDS = [
+    {"id": "remote-001", "title": "Alpha", "completed": False, "version": 1, "updatedAt": 1700000000000},
+    {"id": "remote-002", "title": "Beta", "completed": False, "version": 1, "updatedAt": 1700000000000},
+]
+CREATED = {"id": "device-001", "title": "Gamma", "completed": False, "version": 1, "updatedAt": 1700000300000}
+RENAMED = {**SEEDS[0], "title": "Queued edit", "version": 2, "updatedAt": 1700000301000}
+COMPLETED = {**RENAMED, "completed": True, "version": 3, "updatedAt": 1700000302000}
+FINAL = [CREATED, COMPLETED]
+INTENTS = [
+    {"sequence": 1, "kind": "create", "itemId": "device-001", "payload": {"id": "device-001", "title": "Gamma", "completed": False}},
+    {"sequence": 2, "kind": "rename", "itemId": "remote-001", "payload": {"title": "Queued edit"}},
+    {"sequence": 3, "kind": "toggle", "itemId": "remote-001", "payload": {"completed": True}},
+    {"sequence": 4, "kind": "delete", "itemId": "remote-002", "payload": None},
+]
+
+
+def expected_trace(baseline):
+    def event(method, path, body, status, response):
+        return {"method": method, "path": path, "body": body, "status": status, "response": response}
+    def get(items):
+        return event("GET", "/items", None, 200, {"items": items})
+    if baseline:
+        return [get(SEEDS), get(SEEDS)]
+    return [get(SEEDS),
+            event("POST", "/items", INTENTS[0]["payload"], 201, {"item": CREATED}),
+            event("PATCH", "/items/remote-001", INTENTS[1]["payload"], 200, {"item": RENAMED}),
+            event("PATCH", "/items/remote-001", INTENTS[2]["payload"], 200, {"item": COMPLETED}),
+            event("DELETE", "/items/remote-002", None, 200, {"deletedId": "remote-002"}),
+            get(FINAL)]
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
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "host_pid": os.getpid(), "serial": args.serial,
+              "apk_sha256": hashlib.sha256(Path(args.apk).read_bytes()).hexdigest(),
+              "harness_sha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+              "fixture_sha256": hashlib.sha256((project / "fixture/server.cjs").read_bytes()).hexdigest(),
+              "inputs_sha256": hashlib.sha256((project / "verification/M05-inputs.json").read_bytes()).hexdigest()}
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
+        method = "GET" if body is None else "POST"
+        request = Request(URL + path, data=None if body is None else json.dumps(body).encode(),
+                          headers={"Content-Type": "application/json"}, method=method)
+        with urlopen(request, timeout=3) as response:
+            value = json.load(response)
+            controls.append({"method": method, "path": path, "body": body, "status": response.status, "response": value})
+        (evidence / "http-controls.json").write_text(json.dumps(controls, indent=2) + "\n")
+        return value
+
+    def snapshot(name=None):
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m05-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m05-ui.xml")
+        if name:
+            (evidence / f"{name}.xml").write_text(xml)
+            (evidence / f"{name}.png").write_bytes(adb("exec-out", "screencap", "-p", binary=True))
+        return ET.fromstring(xml)
+
+    def scroll(root, down):
+        container = next((node for node in root.iter("node") if node.get("scrollable") == "true"), None)
+        if container is None:
+            return
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", container.get("bounds")))
+        start, end = (y2 - 40, y1 + 40) if down else (y1 + 40, y2 - 40)
+        adb("shell", "input", "swipe", str((x1 + x2) // 2), str(start), str((x1 + x2) // 2), str(end), "300")
+
+    def find(label, attribute="content-desc", scrollable=False):
+        deadline, attempt = time.monotonic() + 20, 0
+        while time.monotonic() < deadline:
+            root = snapshot()
+            for node in root.iter("node"):
+                if node.get(attribute) == label:
+                    return node
+            if scrollable:
+                scroll(root, attempt % 4 < 2)
+            attempt += 1
+        raise AssertionError(f"Missing {attribute}: {label}")
+
+    def tap(label):
+        node = find(label, scrollable=label.startswith(("Edit ", "Delete ", "Mark ")))
+        assert node.get("enabled") != "false", f"Disabled control: {label}"
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", node.get("bounds")))
+        adb("shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
+
+    def title(label, value):
+        node = find(label)
+        previous = node.get("text", "")
+        tap(label)
+        adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        if previous and previous != "Item title":
+            adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len(previous)))
+        adb("shell", "input", "text", value.replace(" ", "%s"))
+        adb("shell", "input", "keyevent", "KEYCODE_BACK")
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
+            assert schema == (2 if args.baseline else 3), schema
+            items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
+                     for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
+            tables = [row[0] for row in db.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
+            if args.baseline:
+                assert "pending_mutations" not in tables, tables
+                pending = None
+            else:
+                pending = [{"sequence": row[0], "kind": row[1], "itemId": row[2],
+                            "payload": None if row[3] is None else json.loads(row[3])}
+                           for row in db.execute("SELECT sequence, kind, item_id, payload FROM pending_mutations ORDER BY sequence")]
+            value = {"schema_version": schema, "items": items, "pending": pending,
+                     "last_successful_refresh_at": db.execute("SELECT last_successful_refresh_at FROM sync_metadata WHERE singleton=1").fetchone()[0],
+                     "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+        (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
+        return value
+
+    def assert_ui(name, items, pending_count, status):
+        find("Local storage ready")
+        find(f"Item count: {len(items)}")
+        find(f"Sync status: {status}")
+        labels = [node.get("content-desc", "") for node in snapshot(name + "-status").iter("node")]
+        if args.baseline:
+            assert not any(label.startswith("Pending uploads:") for label in labels)
+        else:
+            assert f"Pending uploads: {pending_count}" in labels, labels
+        for item in items:
+            find(item["title"], "text", scrollable=True)
+            find(f"item-row-{item['id']}", "resource-id", scrollable=True)
+            label = f"Mark {item['title']} {'incomplete' if item['completed'] else 'complete'}"
+            assert find(label, scrollable=True).get("checked") == str(item["completed"]).lower()
+        snapshot(name + "-list")
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
+            assert time.monotonic() < deadline, f"Network transition did not reach {expected}: {actual}"
+            time.sleep(0.1)
+        (evidence / f"{name}-connectivity.txt").write_text(connectivity)
+        result[name + "_network"] = actual
+        save()
+
+    def launch():
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--ez", "m03FixedIdentity", "true")
+        find("Local storage ready")
+
+    def completed_operation(index):
+        find("Local storage ready")
+        value = database(f"offline-operation-{index}")
+        if not args.baseline:
+            find(f"Pending uploads: {index}")
+            assert value["pending"] == INTENTS[:index], value
+        assert remote() == {"items": SEEDS, "nextTimestamp": 1700000300000, "requests": expected_trace(True)[:1]}
+        return value
+
+    fixture, original_network = None, None
+    fixture_log = (evidence / "fixture.log").open("w")
+    try:
+        fixture_command = [args.node, str(project / "fixture/server.cjs")]
+        fixture = subprocess.Popen(fixture_command, stdout=fixture_log, stderr=subprocess.STDOUT)
+        result["fixture_command"], result["fixture_pid"] = fixture_command, fixture.pid
+        save()
+        deadline = time.monotonic() + 10
+        while f'"pid":{fixture.pid},' not in (evidence / "fixture.log").read_text():
+            assert fixture.poll() is None, "Owned fixture failed to start"
+            assert time.monotonic() < deadline, "Owned fixture did not become ready"
+            time.sleep(0.05)
+        assert remote() == {"items": SEEDS, "nextTimestamp": 1700000100000, "requests": []}
+        assert remote("/__mutation-clock", {"nextTimestamp": 1700000300000}) == {"nextTimestamp": 1700000300000, "delayMs": 0}
+        original_network = network_state()
+        result["network_before"] = original_network
+        assert original_network == {"airplane_mode_on": "0", "wifi_on": "1", "mobile_data": "1"}, original_network
+        adb("install", "-r", str(Path(args.apk).resolve()))
+        adb("shell", "pm", "clear", PACKAGE)
+        launch()
+        find("Item count: 0")
+        assert database("empty")["items"] == []
+        assert remote()["requests"] == [], "Startup must not fetch"
+        tap("Synchronize")
+        assert_ui("seeded", SEEDS, 0, "fresh")
+        assert database("seeded")["items"] == SEEDS
+        assert remote()["requests"] == expected_trace(True)[:1]
+        set_network(OFFLINE, "offline")
+
+        title("New item title", "Gamma")
+        tap("Add item")
+        find("Item count: 3")
+        completed_operation(1)
+        tap("Edit Alpha")
+        title("Edit item title", "Queued edit")
+        tap("Save title")
+        find("Queued edit", "text", scrollable=True)
+        completed_operation(2)
+        tap("Mark Queued edit complete")
+        assert find("Mark Queued edit incomplete", scrollable=True).get("checked") == "true"
+        completed_operation(3)
+        tap("Delete Beta")
+        find("Item count: 2")
+        result["before_kill"] = completed_operation(4)
+        local = result["before_kill"]["items"]
+        assert [{key: value for key, value in item.items() if key != "updatedAt"} for item in local] == [
+            {key: value for key, value in item.items() if key != "updatedAt"} for item in FINAL]
+        assert all(isinstance(item["updatedAt"], int) for item in local)
+        assert result["before_kill"]["next_id"] == 2
+        assert_ui("before-kill", local, 4, "stale")
+        result["death_boundary_start_command_index"] = len(commands)
+        result["pid_before"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_before"]
+        adb("shell", "am", "force-stop", PACKAGE)
+        result["pid_after_kill"] = adb("shell", "pidof", PACKAGE, check=False)
+        assert not result["pid_after_kill"]
+        assert database("after-kill") == result["before_kill"]
+        assert network_state() == OFFLINE
+        launch()
+        result["pid_after_restart"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_after_restart"] and result["pid_after_restart"] != result["pid_before"]
+        assert network_state() == OFFLINE
+        result["after_restart"] = database("after-restart")
+        assert result["after_restart"] == result["before_kill"]
+        assert_ui("after-restart", local, 4, "stale")
+        assert remote()["requests"] == expected_trace(True)[:1], "Offline restart must not fetch"
+        result["death_boundary_end_command_index"] = len(commands)
+        assert os.getpid() == result["host_pid"]
+        save()
+
+        set_network(original_network, "reconnected")
+        tap("Synchronize")
+        expected = SEEDS if args.baseline else FINAL
+        assert_ui("after-drain", expected, 0, "fresh")
+        result["after_drain"] = database("after-drain")
+        assert result["after_drain"]["items"] == expected
+        if not args.baseline:
+            assert result["after_drain"]["pending"] == []
+        result["remote"] = remote()
+        assert result["remote"] == {"items": expected, "nextTimestamp": 1700000300000 if args.baseline else 1700000304000,
+                                    "requests": expected_trace(args.baseline)}
+        (evidence / "remote-final.json").write_text(json.dumps(result["remote"], indent=2) + "\n")
+        result["accepted_operations"] = [event for event in result["remote"]["requests"] if event["method"] != "GET"]
+        result["status"] = "BASELINE_LIMIT_REPRODUCED" if args.baseline else "PASS"
+        if args.baseline:
+            result["limitation"] = "Local edits survive actual offline process death, but no durable upload intent or pending label exists; next sync pulls unchanged seeds and discards all four unsent edits."
+    except Exception as error:
+        result["status"], result["error"] = "FAIL", repr(error)
+    finally:
+        if original_network is not None:
+            try:
+                set_network(original_network, "restored")
+                result["network_after"] = network_state()
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
+        raise AssertionError(result.get("error", result.get("cleanup_error", "M05 did not complete")))
+
+
+if __name__ == "__main__":
+    main()
diff --git a/verification/M05-inputs.json b/verification/M05-inputs.json
new file mode 100644
index 0000000..6e1cfb0
--- /dev/null
+++ b/verification/M05-inputs.json
@@ -0,0 +1,51 @@
+{
+  "thread": "M05",
+  "specRevision": "ed7baa246ee947c6778e0f84751c9b91cec7abfe",
+  "start": "66d1c646cd99cc392011b773c8221c93a659b1b7",
+  "fixturePort": 18081,
+  "delayMs": 0,
+  "seeds": [
+    {"id": "remote-001", "title": "Alpha", "completed": false, "version": 1, "updatedAt": 1700000000000},
+    {"id": "remote-002", "title": "Beta", "completed": false, "version": 1, "updatedAt": 1700000000000}
+  ],
+  "nextAcceptedMutationTimestamp": 1700000300000,
+  "acceptedTimestampStepMs": 1000,
+  "mutations": [
+    {"kind": "create", "itemId": "device-001", "payload": {"id": "device-001", "title": "Gamma", "completed": false}},
+    {"kind": "rename", "itemId": "remote-001", "payload": {"title": "Queued edit"}},
+    {"kind": "toggle", "itemId": "remote-001", "payload": {"completed": true}},
+    {"kind": "delete", "itemId": "remote-002", "payload": null}
+  ],
+  "final": [
+    {"id": "device-001", "title": "Gamma", "completed": false, "version": 1, "updatedAt": 1700000300000},
+    {"id": "remote-001", "title": "Queued edit", "completed": true, "version": 3, "updatedAt": 1700000302000}
+  ],
+  "android": {
+    "serial": "emulator-5554",
+    "profile": "API34 Pixel6 Google APIs ARM64",
+    "offlineSettings": {"airplane_mode_on": "1", "wifi_on": "0", "mobile_data": "0"},
+    "offlineProof": "dumpsys connectivity contains Active default network: none",
+    "identityMechanism": "Existing BuildConfig.DEBUG m03FixedIdentity launch prop fixes prefix=device; no app code is changed for baseline",
+    "localEditTimes": "Real device clock, checked for integer values and preserved byte-for-byte over restart; remote accepted timestamps are fixed independently",
+    "uiWaitSeconds": 20,
+    "networkTransitionWaitSeconds": 10,
+    "networkPollSeconds": 0.1,
+    "fixtureStartWaitSeconds": 10,
+    "fixturePollSeconds": 0.05,
+    "httpTimeoutSeconds": 3,
+    "adbCommandTimeoutSeconds": 60,
+    "fixtureCleanupWaitSeconds": 10,
+    "deathBoundary": "After all four local operations report completion: capture DB+queue/PID, force-stop, confirm absent PID, relaunch still offline, confirm different PID and exact prior DB+queue. No install, clear, DB rewrite or mutation within boundary.",
+    "drain": "Restore original network, tap Synchronize exactly once; no injected remote failure"
+  },
+  "hostSupportingCases": [
+    "Exact four offline operations at host local times 1700000300000,1700000301000,1700000302000; close SQLite handles and create a new sync object; restore transport and drain once with exact accepted operation trace.",
+    "Fail each local item INSERT/UPDATE/DELETE, the pending intent INSERT, and transaction END for all four operation kinds. Assert prior items, queue and identity survive SQLite reopen unchanged. Retry each failed mutation once at the same fixed inputs.",
+    "No-op blank create, blank rename, same-title rename and missing-item update/delete append no queue entry.",
+    "Upgrade literal schema-v2 seeds with allocator3 and last-success1700000200000, preserving all fields; queue starts empty. Fail queue table creation and assert v2 remains intact before successful reopen.",
+    "Continue the literal v1 migration regression at current schema3, and reject unsupported schema4 without modifying existing Items.",
+    "UI offline edit failure preserves draft/list/pending count; restart re-reads pending count from SQLite with no request; successful foreground drain shows empty pending queue.",
+    "One explicit offline sync failure retains all four ordered intents; a pending queue prevents snapshot replacement from erasing local mutations. No automatic retries or concurrency framework."
+  ],
+  "scope": "Durable ordered foreground upload intent only; no idempotency, remote conflict, crash-safe remote acknowledgement, background work or native module"
+}
diff --git a/verification/M05.md b/verification/M05.md
new file mode 100644
index 0000000..a14e096
--- /dev/null
+++ b/verification/M05.md
@@ -0,0 +1,92 @@
+# M05 verification — attempt 1
+
+- Branch: `track/react-native`; START: `66d1c646cd99cc392011b773c8221c93a659b1b7`.
+- Spec-Revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`.
+- Shared frozen scenario SHA-256: `e9debf71d1d61b9dcdcec2387a15dc150afb91fb84a3ba3680809cc639179604`.
+- Supporting inputs: `verification/M05-inputs.json`, SHA-256 `a200572a3f0b7f2c9b79f4c91da7ca4069272f7de0b1662335dde2b84447208b`.
+- Raw evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/`.
+- Android: one API34 Pixel6 Google APIs ARM64 emulator, `emulator-5554`.
+
+## Unchanged M04 reproduction
+
+Read the full main README, every spec Markdown document, protocol and frozen M05
+scenario, then only this track's implementation/history. The worktree was clean;
+branch, HEAD and peeled `progress/react-native/M04` all matched START. All 43
+tracked files are preserved under `baseline-source/`, with Git-object-verified
+hashes in `baseline-manifest.json`. No application production code changed before
+the baseline Android and desired-result host tests.
+
+`m04-baseline.apk` is byte-identical to main's independently built and verified
+M04 artifact, SHA-256
+`387e3d33cf092a9287ae2d40c2f648006f1fc37feae33dda15f52f8a5c4b2309`.
+The first copy audit (`snapshot-01`, exit1) incorrectly compared it to the earlier
+agent build `befae993950e4d0ee049c80707da4f158acdb8f9601abd721c2da78782c60964`.
+The raw assertion is retained. Main confirmed the distinct independently tested
+artifact, and the corrected read-only manifest audit passed. No rebuild or product
+change was used to resolve this archive-identification mistake.
+
+Before first execution, `frozen-inputs-before-baseline.json` and `frozen-source/`
+captured the harness, inputs, fixture and desired-result test. The fixture adds
+only `/__mutation-clock` to select the required next timestamp1700000300000;
+existing seeds, zero delay and M03/M04 responses remain unchanged. Harness SHA-256
+is `be8f07d6bb79c7e8de998d3d33090d86417c346e1392b1d5c85bcc3f0c2b3c0f`;
+fixture SHA-256 is `46c2a4a1beeef4e50e1826586fa2d48ff63bcd68fda58c1eadfcd72465e828ae`.
+
+Under the explicit baseline device lease, from the branch root:
+
+```sh
+python3 scripts/verify_m05.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/m04-baseline.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M05/baseline-android-01 --baseline
+```
+
+`baseline-android-01` exited0, `BASELINE_LIMIT_REPRODUCED`, in179.108 seconds with
+237 recorded adb commands. The actual RN UI completed create Gamma, rename Alpha
+to Queued edit, complete it, and delete Beta while airplane/Wi-Fi/data were1/0/0
+and `dumpsys connectivity` showed no active default network. SQLite retained both
+edited rows, but schema2 had no pending intent table and the UI no pending label.
+
+Host PID58052 remained alive across app PID2935 → absent → PID4290. The native
+database, all Item fields and allocator2 were identical before death, while dead,
+and after relaunch still offline. No install/clear/database rewrite happened
+between command indexes132 and173. Startup issued no request. The one reconnect
+sync accepted **zero mutations** and pulled the two original seeds, replacing the
+four offline edits. Both HTTP events were GET200. This is the M04 guarantee's
+limit, not damage introduced for reproduction.
+
+The harness restored recorded network0/1/1 and an active default network, stopped
+the RN app and confirmed its PID absent. Its owned fixture PID58066 was stopped
+with `Popen.terminate()` and exited0. The lease was released immediately after
+reading those cleanup results; no further device command ran under that lease.
+
+The unchanged desired-result test also fails on M04 after SQLite reopening:
+
+```sh
+env PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH /opt/homebrew/bin/npm test -- --runInBand --watchman=false __tests__/sync.test.ts -t 'M05 fixed four offline mutations'
+```
+
+`baseline-jest-01` failed before the scenario because sandboxing denied the
+127.0.0.1:18081 listener (`EPERM`). The same test was retried with proper sandbox
+approval as `baseline-jest-02`; exit1 is the expected desired-result failure:
+after the new session's drain, local/remote contain Alpha/Beta instead of the
+specified Gamma/Queued edit. Its log records the exact two-GET trace and preserved
+pre-restart edits. Five unrelated tests were skipped. Assertions and inputs are
+not weakened after this failure.
+
+## Invocation ledger at reproduction
+
+Each runner invocation has an immutable `.command.json` (argv, cwd, selected
+runtime environment, source hashes, exit and duration), `.log`, and source snapshot
+under `snapshots/<invocation>/`. All repeated or failed invocations remain present.
+
+| Invocation | Outcome | Exit |
+|---|---|---:|
+| snapshot-01 | Archive hash expectation mismatch described above; source/APK preserved | 1 |
+| corrected manifest audit | All43 baseline files match START; current APK matches main's verified artifact | 0 |
+| harness-syntax-01 | `python3 -m py_compile scripts/verify_m05.py` | 0 |
+| fixture-syntax-01 | `node --check fixture/server.cjs` | 0 |
+| baseline-typecheck-01 | `npm run typecheck`, unchanged application source | 0 |
+| baseline-android-01 | Real offline edits/process death/drain; baseline limit and cleanup verified | 0 |
+| baseline-jest-01 | Listener denied by sandbox; no scenario result | 1 |
+| baseline-jest-02 | Expected exact-final-state failure on unchanged M04 | 1 |
+
+This reproduction commit is intentionally red. Implementation and post-fix
+verification remain pending at this point in history.


