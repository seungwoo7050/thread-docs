# M04 — Offline Read

## `test: reproduce missing offline refresh status`

diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index e3233af..509c97b 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -1,8 +1,9 @@
 import React from 'react';
-import {fireEvent, render, screen, waitFor} from '@testing-library/react-native';
+import {act, fireEvent, render, screen, waitFor} from '@testing-library/react-native';
 import App from '../src/App';
 import {openItemStore} from '../src/itemStore';
 import {closeDatabases, failNextSql} from './sqliteNative';
+import {ForegroundSync, JsonRequest} from '../src/sync';
 
 const saved = () => waitFor(() => expect(screen.getByLabelText('Local storage ready')).toBeTruthy());
 
@@ -97,3 +98,140 @@ test('M03 explicit sync publishes a committed database read, and never a service
   expect(screen.getByText('Alpha')).toBeTruthy();
   expect(screen.getByTestId('item-row-remote-001')).toBeTruthy();
 });
+
+const m04Seeds = [
+  {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
+  {id: 'remote-002', title: 'Beta', completed: false, version: 1, updatedAt: 1700000000000},
+];
+const m04Final = [{...m04Seeds[0], title: 'Remote revised', version: 2, updatedAt: 1700000201000}, m04Seeds[1]];
+
+test('M04 fixed offline and HTTP failures preserve the list and last success before explicit reconnect', async () => {
+  let clock = 1700000200000;
+  let offline = false;
+  let getFailures = 0;
+  let canonical = m04Seeds;
+  const responses: number[] = [];
+  const request: JsonRequest = jest.fn(async (_url, options) => {
+    if (offline) {throw new TypeError('Network request failed');}
+    expect(options.method).toBe('GET');
+    if (getFailures > 0) {
+      getFailures -= 1;
+      responses.push(503);
+      return {status: 503, json: async () => ({error: 'Temporary GET failure'})};
+    }
+    responses.push(200);
+    return {status: 200, json: async () => ({items: canonical})};
+  });
+  const store = await openItemStore();
+  const session = new ForegroundSync(store, 'http://fixed-m04', request, 'device', () => clock);
+  const view = render(<App openStore={async () => store} createSync={() => session} />);
+  await saved();
+  expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(await store.read()).toEqual(m04Seeds);
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+
+  offline = true;
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(screen.getByLabelText('Sync status: error')).toBeTruthy());
+  await saved();
+  expect(screen.getByText('Could not refresh: Network request failed')).toBeTruthy();
+  expect(screen.queryByText('Change not saved')).toBeNull();
+  expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
+  expect(screen.getByText('Alpha')).toBeTruthy();
+  expect(screen.getByText('Beta')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+  expect(await store.read()).toEqual(m04Seeds);
+  expect(responses).toEqual([200]);
+
+  offline = false;
+  getFailures = 1;
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(screen.getByText('Could not refresh: GET /items failed (HTTP 503)')).toBeTruthy());
+  await saved();
+  expect(screen.getByLabelText('Sync status: error')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+  expect(await store.read()).toEqual(m04Seeds);
+  expect(responses).toEqual([200, 503]);
+
+  offline = true;
+  canonical = m04Final;
+  expect(await store.read()).toEqual(m04Seeds);
+  expect(responses).toEqual([200, 503]);
+  offline = false;
+  getFailures = 0;
+  clock = 1700000202000;
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000202000')).toBeTruthy();
+  expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
+  expect(screen.getByText('Remote revised')).toBeTruthy();
+  expect(screen.getByText('Beta')).toBeTruthy();
+  expect(screen.queryByText('Alpha')).toBeNull();
+  expect(await store.read()).toEqual(m04Final);
+  expect(responses).toEqual([200, 503, 200]);
+  expect(request).toHaveBeenCalledTimes(4);
+
+  view.unmount();
+  closeDatabases();
+  offline = true;
+  const reopened = await openItemStore();
+  render(<App openStore={async () => reopened}
+    createSync={() => new ForegroundSync(reopened, 'http://fixed-m04', request, 'device', () => clock)} />);
+  await saved();
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000202000')).toBeTruthy();
+  expect(screen.getByText('Remote revised')).toBeTruthy();
+  expect(screen.getByText('Beta')).toBeTruthy();
+  expect(request).toHaveBeenCalledTimes(4);
+});
+
+test('M04 cached startup and an unresolved refresh never block the saved list', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot(m04Seeds, 1700000200000);
+  let resolveRequest: (value: Awaited<ReturnType<JsonRequest>>) => void = () => {throw new Error('GET not started');};
+  const request: JsonRequest = jest.fn(() => new Promise(resolve => {resolveRequest = resolve;}));
+  render(<App openStore={async () => store}
+    createSync={() => new ForegroundSync(store, 'http://fixed-m04', request, 'device', () => 1700000202000)} />);
+  await saved();
+  expect(request).not.toHaveBeenCalled();
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+  expect(screen.getByText('Alpha')).toBeTruthy();
+  expect(screen.getByText('Beta')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(request).toHaveBeenCalledTimes(1));
+  expect(screen.getByLabelText('Sync status: refreshing')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+  expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
+  expect(screen.getByText('Alpha')).toBeTruthy();
+  expect(screen.getByText('Beta')).toBeTruthy();
+  await act(async () => {resolveRequest({status: 200, json: async () => ({items: m04Seeds})});});
+  await saved();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000202000')).toBeTruthy();
+});
+
+test('M04 a committed local edit becomes stale without changing last successful refresh time', async () => {
+  const store = await openItemStore();
+  const request: JsonRequest = async () => ({status: 200, json: async () => ({items: m04Seeds})});
+  render(<App openStore={async () => store}
+    createSync={() => new ForegroundSync(store, 'http://fixed-m04', request, 'device', () => 1700000200000)} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Edit Alpha'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Locally revised');
+  const clock = jest.spyOn(Date, 'now').mockReturnValue(1700000202000);
+  try {
+    fireEvent.press(screen.getByLabelText('Save title'));
+    await saved();
+  } finally {clock.mockRestore();}
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
+  expect(await store.read()).toEqual([{...m04Seeds[0], title: 'Locally revised', version: 2, updatedAt: 1700000202000}, m04Seeds[1]]);
+});
diff --git a/fixture/server.cjs b/fixture/server.cjs
index d25f476..42785d4 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03 fixture, not a backend service.
+// A disposable deterministic M03/M04 fixture, not a backend service.
 const http = require('node:http');
 
 const SEEDS = [
@@ -10,11 +10,13 @@ function createFixture() {
   let items;
   let nextTimestamp;
   let requests;
+  let getFailures;
   const snapshot = () => [...items.values()].sort((a, b) => a.id.localeCompare(b.id));
   function reset() {
     items = new Map(SEEDS.map(item => [item.id, {...item}]));
     nextTimestamp = 1700000100000;
     requests = [];
+    getFailures = 0;
   }
   reset();
   const state = () => ({items: snapshot(), nextTimestamp, requests});
@@ -35,7 +37,29 @@ function createFixture() {
       if (raw) {body = JSON.parse(raw);}
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
-      if (method === 'GET' && path === '/items') {return reply(200, {items: snapshot()});}
+      // M04 controls affect only the explicitly selected GET or remote Item.
+      // Every response still has zero artificial delay; there is no retry loop.
+      if (method === 'POST' && path === '/__get-failures') {
+        if (!body || !Number.isSafeInteger(body.count) || body.count < 0) {
+          return reply(400, {error: 'Nonnegative GET failure count is required'});
+        }
+        getFailures = body.count;
+        return reply(200, {getFailures, delayMs: 0});
+      }
+      if (method === 'POST' && path === '/__remote-change') {
+        const item = items.get(body?.id);
+        if (!item || typeof body.title !== 'string' || !body.title.trim()
+            || !Number.isSafeInteger(body.updatedAt)) {
+          return reply(400, {error: 'Existing id, title and fixed updatedAt are required'});
+        }
+        const updated = {...item, title: body.title.trim(), version: item.version + 1, updatedAt: body.updatedAt};
+        items.set(item.id, updated);
+        return reply(200, {item: updated});
+      }
+      if (method === 'GET' && path === '/items') {
+        if (getFailures > 0) {getFailures -= 1; return reply(503, {error: 'Temporary GET failure'});}
+        return reply(200, {items: snapshot()});
+      }
       if (method === 'POST' && path === '/items') {
         if (!body || typeof body.id !== 'string' || !/^[a-zA-Z0-9_-]+$/.test(body.id)
             || typeof body.title !== 'string' || !body.title.trim() || typeof body.completed !== 'boolean') {
diff --git a/scripts/verify_m04.py b/scripts/verify_m04.py
new file mode 100644
index 0000000..32ac942
--- /dev/null
+++ b/scripts/verify_m04.py
@@ -0,0 +1,291 @@
+#!/usr/bin/env python3
+"""M04 fixed real Android offline/HTTP-failure/reconnect scenario.
+
+Requires an exclusive emulator lease. Owns its fixture and restores all changed
+connectivity settings. --baseline records the unchanged M03 presentation limit.
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
+SEEDS = [
+    {"id": "remote-001", "title": "Alpha", "completed": False, "version": 1, "updatedAt": 1700000000000},
+    {"id": "remote-002", "title": "Beta", "completed": False, "version": 1, "updatedAt": 1700000000000},
+]
+REVISED = {**SEEDS[0], "title": "Remote revised", "version": 2, "updatedAt": 1700000201000}
+FINAL = [REVISED, SEEDS[1]]
+FIRST_REFRESH = 1700000200000
+FINAL_REFRESH = 1700000202000
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
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "host_pid": os.getpid(), "serial": args.serial,
+              "apk_sha256": hashlib.sha256(Path(args.apk).read_bytes()).hexdigest(),
+              "harness_sha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+              "fixture_sha256": hashlib.sha256((project / "fixture/server.cjs").read_bytes()).hexdigest(),
+              "inputs_sha256": hashlib.sha256((project / "verification/M04-inputs.json").read_bytes()).hexdigest()}
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
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m04-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m04-ui.xml")
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
+        node = find(label)
+        assert node.get("enabled") != "false", f"Disabled control: {label}"
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", node.get("bounds")))
+        adb("shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
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
+            assert schema == (1 if args.baseline else 2), schema
+            assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
+            items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
+                     for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
+            last = None if args.baseline else db.execute("SELECT last_successful_refresh_at FROM sync_metadata WHERE singleton=1").fetchone()[0]
+            value = {"schema_version": schema, "items": items, "last_successful_refresh_at": last,
+                     "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+        (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
+        return value
+
+    def assert_ui(name, items, status, last, failure=None):
+        find("Local storage error" if args.baseline and failure else "Local storage ready")
+        find("Item count: 2")
+        root = snapshot(name + "-status")
+        nodes = list(root.iter("node"))
+        labels = [node.get("content-desc", "") for node in nodes]
+        texts = [node.get("text", "") for node in nodes]
+        if args.baseline:
+            assert not any(label.startswith(("Sync status:", "Last successful refresh:")) for label in labels)
+            if failure:
+                assert "Change not saved" in texts
+                assert any("Could not synchronize:" in text and failure in text for text in texts), texts
+        else:
+            assert f"Sync status: {status}" in labels, labels
+            assert f"Last successful refresh: {last}" in labels, labels
+            assert "Change not saved" not in texts
+            if failure:
+                assert any("Could not refresh:" in text and failure in text for text in texts), texts
+        for item in items:
+            find(item["title"], "text", scrollable=True)
+            find(f"item-row-{item['id']}", "resource-id", scrollable=True)
+            assert find(f"Mark {item['title']} complete", scrollable=True).get("checked") == "false"
+        snapshot(name + "-list")
+        value = database(name)
+        assert value["items"] == items, value
+        assert value["next_id"] == 1, value
+        assert value["last_successful_refresh_at"] == (None if args.baseline else last), value
+        result[name] = {**value, "visible_status": "missing" if args.baseline else status,
+                        "local_status": "error" if args.baseline and failure else "ready"}
+        save()
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
+        # M03 ignores this extra. M04 exposes it only in a debug build.
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--ez", "m04FixedClock", "true")
+        find("Local storage ready")
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
+        original_network = network_state()
+        result["network_before"] = original_network
+        assert original_network == {"airplane_mode_on": "0", "wifi_on": "1", "mobile_data": "1"}, original_network
+        adb("install", "-r", str(Path(args.apk).resolve()))
+        adb("shell", "pm", "clear", PACKAGE)
+        launch()
+        find("Item count: 0")
+        assert database("empty")["items"] == []
+        assert remote()["requests"] == [], "Startup must not fetch"
+        if not args.baseline:
+            find("Sync status: stale")
+            find("Last successful refresh: never")
+
+        tap("Synchronize")
+        assert_ui("first-refresh", SEEDS, "fresh", FIRST_REFRESH)
+        assert len(remote()["requests"]) == 1
+        set_network(OFFLINE, "offline")
+        tap("Synchronize")
+        assert_ui("offline-error", SEEDS, "error", FIRST_REFRESH, "Network request failed")
+        assert len(remote()["requests"]) == 1, "Offline refresh must fail before a remote response"
+
+        set_network(original_network, "online-for-http-failure")
+        assert remote("/__get-failures", {"count": 1}) == {"getFailures": 1, "delayMs": 0}
+        tap("Synchronize")
+        assert_ui("http-error", SEEDS, "error", FIRST_REFRESH, "HTTP 503")
+        assert len(remote()["requests"]) == 2
+        set_network(OFFLINE, "offline-for-remote-change")
+        assert remote("/__remote-change", {"id": "remote-001", "title": "Remote revised", "updatedAt": 1700000201000}) == {"item": REVISED}
+        assert database("while-remote-changed")["items"] == SEEDS
+        assert len(remote()["requests"]) == 2
+        set_network(original_network, "reconnected")
+        assert remote("/__get-failures", {"count": 0}) == {"getFailures": 0, "delayMs": 0}
+        tap("Synchronize")
+        assert_ui("reconciled", FINAL, "fresh", FINAL_REFRESH)
+        expected_trace = [
+            {"method": "GET", "path": "/items", "body": None, "status": 200, "response": {"items": SEEDS}},
+            {"method": "GET", "path": "/items", "body": None, "status": 503, "response": {"error": "Temporary GET failure"}},
+            {"method": "GET", "path": "/items", "body": None, "status": 200, "response": {"items": FINAL}},
+        ]
+        result["remote"] = remote()
+        assert result["remote"] == {"items": FINAL, "nextTimestamp": 1700000100000, "requests": expected_trace}
+        (evidence / "remote-final.json").write_text(json.dumps(result["remote"], indent=2) + "\n")
+        result["fixed_sequence_complete_command_index"] = len(commands)
+        save()
+
+        # Frozen supplemental local-first read: no clear/reinstall/data mutation.
+        set_network(OFFLINE, "offline-restart")
+        result["pid_before"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_before"]
+        adb("shell", "am", "force-stop", PACKAGE)
+        result["pid_after_kill"] = adb("shell", "pidof", PACKAGE, check=False)
+        assert not result["pid_after_kill"]
+        launch()
+        result["pid_after_restart"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_after_restart"] and result["pid_before"] != result["pid_after_restart"]
+        assert_ui("offline-startup", FINAL, "stale", FINAL_REFRESH)
+        assert remote()["requests"] == expected_trace, "Local-first cold startup must not fetch"
+        assert os.getpid() == result["host_pid"]
+        result["status"] = "BASELINE_LIMIT_REPRODUCED" if args.baseline else "PASS"
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
+        raise AssertionError(result.get("error", result.get("cleanup_error", "M04 did not complete")))
+
+
+if __name__ == "__main__":
+    main()
diff --git a/verification/M04-inputs.json b/verification/M04-inputs.json
new file mode 100644
index 0000000..b3f37ec
--- /dev/null
+++ b/verification/M04-inputs.json
@@ -0,0 +1,39 @@
+{
+  "thread": "M04",
+  "specRevision": "ed7baa246ee947c6778e0f84751c9b91cec7abfe",
+  "start": "8ce636c1aecdb19a098568aa122fd32f239d16a5",
+  "fixturePort": 18081,
+  "delayMs": 0,
+  "seeds": [
+    {"id": "remote-001", "title": "Alpha", "completed": false, "version": 1, "updatedAt": 1700000000000},
+    {"id": "remote-002", "title": "Beta", "completed": false, "version": 1, "updatedAt": 1700000000000}
+  ],
+  "successfulRefreshTimes": [1700000200000, 1700000202000],
+  "sequence": ["reset local storage", "explicit successful refresh", "force transport offline", "explicit failed refresh before response", "restore transport", "next GET HTTP503 once", "explicit failed refresh", "force transport offline", "remote title change", "restore transport and clear GET failure count", "one explicit successful refresh"],
+  "remoteChange": {"id": "remote-001", "title": "Remote revised", "version": 2, "updatedAt": 1700000201000},
+  "localMutations": [],
+  "android": {
+    "serial": "emulator-5554",
+    "profile": "API34 Pixel6 Google APIs ARM64",
+    "offlineMechanism": "airplane mode enabled, Wi-Fi and mobile data disabled; verify Active default network: none",
+    "clockMechanism": "BuildConfig.DEBUG gated initial prop; consume the two fixed timestamps only after a valid HTTP snapshot is received",
+    "baseline": "unchanged verified M03 APK; it has no refresh clock or status, so record their absence rather than claiming fixed clock injection",
+    "uiWaitSeconds": 20,
+    "networkTransitionWaitSeconds": 10,
+    "networkPollSeconds": 0.1,
+    "fixtureStartWaitSeconds": 10,
+    "fixturePollSeconds": 0.05,
+    "httpTimeoutSeconds": 3,
+    "adbCommandTimeoutSeconds": 60,
+    "fixtureCleanupWaitSeconds": 10,
+    "supplementAfterFixedSequence": "force-stop/relaunch offline without data reset, then check both rows and last-success timestamp; startup must make no HTTP request"
+  },
+  "hostSupportingCases": [
+    "Hold the GET promise unresolved after a seeded SQLite open; both rows and stale/refreshing status stay visible before release. No artificial timed delay.",
+    "Fail the snapshot metadata UPDATE after an initial success; both rows and last-success time roll back together, including after SQLite reopen.",
+    "Open a literal schema-v1 SQLite file containing the two seeds and next_id=3; migrate once, preserve all fields and allocator, initialize last-success to null, then reopen.",
+    "Reject schema-v3 without touching Items, replacing the earlier unsupported-next-version test because schema-v2 is now supported.",
+    "On an ordinary local edit after success, status becomes stale while the successful-refresh time remains unchanged. This preserves M03 session-only upload limits."
+  ],
+  "acceptance": "Offline and HTTP503 preserve exact seeds and prior successful-refresh time; errors are separate from local-save status. Final rows are exactly revised remote-001 and unchanged Beta with fresh status. Local startup never waits for network. No retries or background work."
+}
diff --git a/verification/M04.md b/verification/M04.md
new file mode 100644
index 0000000..9d7c494
--- /dev/null
+++ b/verification/M04.md
@@ -0,0 +1,101 @@
+# M04 verification — attempt 1
+
+- Branch: `track/react-native`; START: `8ce636c1aecdb19a098568aa122fd32f239d16a5`.
+- Spec-Revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`.
+- Shared M04 scenario SHA-256: `905cf29e09f4e1f085e3804f86fbe9284a519d825547dea1906702b8bacffd43`.
+- Fixed inputs: `verification/M04-inputs.json`, SHA-256 `e6111c51077ed56e10815d7809e3a9814496a4e907ee53c30b357d1b463bcfae`.
+- Raw evidence: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/`.
+- Android: the same API34 Pixel6 Google APIs ARM64 emulator, `emulator-5554`.
+
+## Reproduction before production edits
+
+Read the complete main README, all spec Markdown, protocol and frozen M04 scenario,
+then only this track's source, history and verification. START matched the peeled
+M03 progress tag and the worktree was clean. No other implementation was inspected.
+
+`baseline-source/` and `baseline-manifest.json` preserve all 40 tracked M03 files
+and their hashes before any edit. `m03-verified-baseline.apk` is byte-identical to
+main's verified M03 APK, SHA-256
+`79e309d347aa475709a1f4f6ad2b637b7dc2d5f5d3f1db3fa5ebe68378904186`.
+Only fixed fixture controls, the external harness, inputs and desired-status tests
+were added before reproduction; application production source was unchanged.
+
+The exact device invocation, from the branch root under an exclusive lease, was:
+
+```sh
+python3 scripts/verify_m04.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/m03-verified-baseline.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M04/baseline-android-01 --baseline
+```
+
+`baseline-android-01` exited 0, `BASELINE_LIMIT_REPRODUCED`, in 135.912 seconds.
+It used actual RN fetch, native SQLite, Android UI controls and OS offline state.
+Both offline and HTTP503 failures retained the exact two seeds. Reconnection
+already produced `Remote revised`/version2/1700000201000 and unchanged Beta.
+The UI had no stale/fresh/status or last-success label; both refresh failures
+incorrectly showed `Local storage error` and `Change not saved`. No production
+behavior was damaged to create the baseline. M03 has no refresh clock to inject
+or timestamp to assert; its absence is recorded, not a fixed-clock success claim.
+
+Only three Item HTTP responses occurred: GET200 with seeds, GET503, GET200 with
+the exact final rows. The offline refresh produced no response or fixture request.
+The supplemental offline restart kept both native rows and made no HTTP request.
+Host PID10419 survived app PID22291 being force-stopped, absent, then relaunched
+as PID24628. The run made no data clear, install or database write across that
+restart boundary. All 264 adb commands, UI XML/PNGs, SQLite files and HTTP controls
+are in `baseline-android-01/`. Airplane/Wi-Fi/mobile data were 1/0/0 offline, with
+`Active default network: none`; cleanup restored the original 0/1/1 and an active
+network. The app was stopped and its absence confirmed before lease release.
+Owned fixture PID10420 was terminated through `Popen.terminate()` and exited 0.
+
+The unchanged desired-status test also ran before the fix:
+
+```sh
+env PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH /opt/homebrew/bin/npm test -- --runInBand --watchman=false __tests__/App.test.tsx -t 'M04 fixed offline'
+```
+
+`baseline-jest-01` exited **1**: one selected test failed exactly at the missing
+`Sync status: fresh` label after it had confirmed both saved seeds; six tests
+were skipped. This expected failure remains raw evidence. Its full source was
+captured before execution and the desired assertions are not weakened afterward.
+
+## Frozen controls and supporting cases
+
+The fixture has zero artificial delay. `/__get-failures` sets only a nonnegative
+GET failure counter; `/__remote-change` revises the selected existing Item and
+increments its version with the exact supplied timestamp. Ordinary M03 state and
+CRUD responses stay unchanged. There is no automatic retry.
+
+The device harness SHA-256 is
+`dcb5e9ec769ece557c468e7516e4bc668a9e74dd8a18e1a5979906df6125d3dd`;
+the fixture SHA-256 is
+`795de077adf64ab25a3f6b08ce6851fa0f8649e45e66856d6d943a462c3db188`.
+Both and all timeouts were frozen before baseline execution. The raw freeze is
+`frozen-inputs-before-baseline.json`; per-invocation source copies live in
+`snapshots/<invocation>/`.
+
+Supporting cases frozen before execution are a held/unresolved GET without a
+timer, an atomic metadata-write rollback, literal schema-v1 migration preserving
+the two seeds and allocator3, rejection of schema-v3, a separate local rename to
+`Locally revised` at1700000202000 after refresh at1700000200000, and an offline
+process restart after the fixed no-local-edits scenario. None changes its inputs,
+order, delay, error point or expected rows based on results.
+
+## Invocation ledger
+
+Each named invocation has `.command.json` (argv, cwd, environment, exits,
+timestamps and source hashes), `.log`, and a pre-execution source snapshot in the
+raw directory. Commands use Node22, JDK17 and the React Native Gradle cache from
+the implementer protocol. Device commands require a fresh exclusive lease.
+
+| Invocation | Command/result | Exit |
+|---|---|---:|
+| harness-syntax-01 | `python3 -m py_compile scripts/verify_m04.py` | 0 |
+| fixture-syntax-01 | `node --check fixture/server.cjs` | 0 |
+| baseline-typecheck-01 | `npm run typecheck`; unchanged application source | 0 |
+| baseline-android-01 | Full unchanged-M03 Android baseline above; limitation reproduced, cleanup verified | 0 |
+| baseline-jest-01 | Desired fresh-state assertion fails on unchanged M03; one failed/six skipped | 1 |
+
+## Current verification boundary
+
+This reproduction commit does not claim a fixed M04 implementation or completed
+verification. Final host/build/Android results and handoff scope are appended
+after implementation. No progress tag or main index record is created here.


