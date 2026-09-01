## `test: verify M03 Android synchronization and restart`

diff --git a/TRACK.md b/TRACK.md
index 8dabfb4..b0612d0 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -26,6 +26,36 @@ No network client, sync state, pending queue, background work, or application
 native module is added. The dependency patch only supplies its AGP namespace and
 removes a legacy manifest namespace and unused null NDK-version setting.
 
+## M03: explicit foreground synchronization
+
+`fixture/server.cjs` is a disposable loopback HTTP fixture on port 18081. It has
+only Item CRUD and test reset/state controls. Seeds, mutation timestamps, zero
+delay and exact response payloads follow the frozen M03 scenario. Android uses
+`10.0.2.2`; cleartext is allowed only for that emulator host address.
+
+The Synchronize button establishes a foreground session by pulling the remote
+canonical snapshot into one native SQLite transaction. Local CRUD still commits
+to SQLite immediately. Later explicit syncs compare those rows with the session's
+last committed shared snapshot, send creates/changed fields/deletes, then pull and
+commit the canonical result. The UI always reads committed SQLite, including after
+sync and process restart; it does not own a separate network result array.
+
+Unconnected M01/M02 operation and IDs stay unchanged. Synchronized creates use a
+session namespace containing time and random nonces plus the durable local
+counter, avoiding a shared fixed per-device namespace. Complete IDs persist with
+Items; no security or cryptographic identity claim is made. `device-001` is only
+the deterministic test value. The Android launch override for that prefix is
+strictly `BuildConfig.DEBUG` gated; it is an initial React prop, not a native module.
+
+This is deliberately a foreground baseline: the last shared snapshot is only in
+memory. A new session's first explicit sync replaces local Items with remote
+canonical state, as the screen warns. Unsent edits have no durable upload intent
+across a session/process restart. A partially completed sync has no automatic
+retry, duplicate protection, conflict policy, or crash-safe continuation. Local
+reads/writes remain persistent, but these upload guarantees arrive in later
+Threads. No queue, background work, push, state framework, or new dependency was
+introduced.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
@@ -45,6 +75,10 @@ adb shell pm clear com.mse.reactnative
 cd ..
 # With the shared device lease granted:
 python3 scripts/verify_m02.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m02
+# M03 harness owns/stops its port-18081 fixture. Use a new evidence directory.
+python3 scripts/verify_m03.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m03
+# To run the fixture separately for manual foreground sync:
+node fixture/server.cjs
 ```
 
 Debug builds bundle their JavaScript and use Hermes with developer support disabled,
diff --git a/scripts/verify_m03.py b/scripts/verify_m03.py
new file mode 100644
index 0000000..f345df3
--- /dev/null
+++ b/scripts/verify_m03.py
@@ -0,0 +1,301 @@
+#!/usr/bin/env python3
+"""M03 actual Android fetch/SQLite/UI verification; requires the device lease.
+
+Owns its loopback fixture process. No connectivity settings are changed. The
+first database's process-death boundary finishes before a second DB is created.
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
+GAMMA = {"id": "device-001", "title": "Gamma", "completed": False, "version": 1, "updatedAt": 1700000100000}
+RENAMED = {**SEEDS[0], "title": "Alpha synced", "version": 2, "updatedAt": 1700000101000}
+COMPLETED = {**RENAMED, "completed": True, "version": 3, "updatedAt": 1700000102000}
+FINAL = [GAMMA, COMPLETED]
+
+
+def expected_trace():
+    def event(method, path, body, status, response):
+        return {"method": method, "path": path, "body": body, "status": status, "response": response}
+    def get(items):
+        return event("GET", "/items", None, 200, {"items": items})
+    return [
+        get(SEEDS),
+        event("POST", "/items", {"id": "device-001", "title": "Gamma", "completed": False}, 201, {"item": GAMMA}),
+        get([GAMMA, *SEEDS]),
+        event("PATCH", "/items/remote-001", {"title": "Alpha synced"}, 200, {"item": RENAMED}),
+        get([GAMMA, RENAMED, SEEDS[1]]),
+        event("PATCH", "/items/remote-001", {"completed": True}, 200, {"item": COMPLETED}),
+        get([GAMMA, COMPLETED, SEEDS[1]]),
+        event("DELETE", "/items/remote-002", None, 200, {"deletedId": "remote-002"}),
+        get(FINAL), get(FINAL),
+    ]
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--node", default="node")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    args = parser.parse_args()
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    project = Path(__file__).resolve().parent.parent
+    commands, http_commands = [], []
+    result = {"status": "RUNNING", "host_pid": os.getpid(), "serial": args.serial,
+              "apk_sha256": hashlib.sha256(Path(args.apk).read_bytes()).hexdigest(),
+              "harness_sha256": hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+              "fixture_sha256": hashlib.sha256((project / "fixture/server.cjs").read_bytes()).hexdigest()}
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
+    def remote(path="/__state", method="GET"):
+        request = Request(URL + path, method=method)
+        with urlopen(request, timeout=3) as response:
+            payload = json.load(response)
+            http_commands.append({"method": method, "url": request.full_url, "status": response.status, "response": payload})
+        (evidence / "http-controls.json").write_text(json.dumps(http_commands, indent=2) + "\n")
+        return payload
+
+    def snapshot(name=None):
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m03-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m03-ui.xml")
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
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", node.get("bounds")))
+        assert node.get("enabled") != "false", f"Disabled control: {label}"
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
+    def database(name, filename="items.db"):
+        files = adb("shell", "run-as", PACKAGE, "ls", "files").splitlines()
+        path = evidence / f"{name}.db"
+        for suffix in ("", "-wal", "-shm"):
+            if filename + suffix in files:
+                Path(str(path) + suffix).write_bytes(adb("exec-out", "run-as", PACKAGE, "cat", f"files/{filename}{suffix}", binary=True))
+        assert path.exists(), "Native SQLite file missing"
+        with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
+            assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+            assert db.execute("PRAGMA user_version").fetchone()[0] == 1
+            assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
+            items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
+                     for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
+            value = {"items": items, "schema_version": 1,
+                     "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+        (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
+        return value
+
+    def synchronize(name, expected, request_count):
+        tap("Synchronize")
+        deadline = time.monotonic() + 20
+        state = remote()
+        while len(state["requests"]) < request_count and time.monotonic() < deadline:
+            time.sleep(0.05)
+            state = remote()
+        assert len(state["requests"]) == request_count, state
+        assert state["items"] == expected, state
+        find("Local storage ready")
+        value = database(name)
+        assert value["items"] == expected, value
+        return value
+
+    def assert_ui(name):
+        # Reveal the top of the short Item list for a complete final screenshot.
+        scroll(snapshot(), False)
+        nodes = list(snapshot(name).iter("node"))
+        assert any(node.get("content-desc") == "Item count: 2" for node in nodes)
+        for item in FINAL:
+            assert any(node.get("text") == item["title"] for node in nodes)
+            assert any(node.get("resource-id") == f"item-row-{item['id']}" for node in nodes)
+        assert any(node.get("content-desc") == "Mark Alpha synced incomplete" and node.get("checked") == "true" for node in nodes)
+        assert any(node.get("content-desc") == "Mark Gamma complete" and node.get("checked") == "false" for node in nodes)
+        assert not any(node.get("text") == "Beta" or node.get("resource-id") == "item-row-remote-002" for node in nodes)
+
+    def network_state():
+        return {key: adb("shell", "settings", "get", "global", key)
+                for key in ("airplane_mode_on", "wifi_on", "mobile_data")}
+
+    fixture = None
+    log = (evidence / "fixture.log").open("w")
+    try:
+        fixture_command = [args.node, str(project / "fixture/server.cjs")]
+        fixture = subprocess.Popen(fixture_command, stdout=log, stderr=subprocess.STDOUT)
+        result["fixture_command"], result["fixture_pid"] = fixture_command, fixture.pid
+        save()
+        deadline = time.monotonic() + 10
+        while True:
+            assert fixture.poll() is None, "Owned fixture failed to start; see fixture.log"
+            # Do not accidentally use an unrelated listener already on the port.
+            if f'"pid":{fixture.pid},' in (evidence / "fixture.log").read_text():
+                break
+            assert time.monotonic() < deadline, "Owned fixture did not become ready"
+            time.sleep(0.05)
+        state = remote()
+        assert state == {"items": SEEDS, "nextTimestamp": 1700000100000, "requests": []}, state
+        result["network_before"] = network_state()
+        adb("install", "-r", str(Path(args.apk).resolve()))
+        adb("shell", "pm", "clear", PACKAGE)
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--ez", "m03FixedIdentity", "true")
+        find("Item count: 0")
+        find("Local storage ready")
+        assert database("first-empty")["items"] == []
+        synchronize("first-pull", SEEDS, 1)
+        find("Alpha", "text")
+        find("Beta", "text")
+        snapshot("first-pull-ui")
+
+        title("New item title", "Gamma")
+        tap("Add item")
+        find("Item count: 3")
+        find("Local storage ready")
+        assert any(item["id"] == "device-001" for item in database("local-create")["items"])
+        synchronize("after-create", [GAMMA, *SEEDS], 3)
+        tap("Edit Alpha")
+        title("Edit item title", "Alpha synced")
+        tap("Save title")
+        find("Alpha synced", "text", scrollable=True)
+        find("Local storage ready")
+        synchronize("after-rename", [GAMMA, RENAMED, SEEDS[1]], 5)
+        tap("Mark Alpha synced complete")
+        assert find("Mark Alpha synced incomplete", scrollable=True).get("checked") == "true"
+        find("Local storage ready")
+        synchronize("after-complete", [GAMMA, COMPLETED, SEEDS[1]], 7)
+        tap("Delete Beta")
+        find("Item count: 2")
+        find("Local storage ready")
+        result["before_kill"] = synchronize("before-kill", FINAL, 9)
+        assert result["before_kill"]["next_id"] == 2
+        assert_ui("before-kill-ui")
+        result["pid_before"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_before"]
+        adb("shell", "am", "force-stop", PACKAGE)
+        result["pid_after_kill"] = adb("shell", "pidof", PACKAGE, check=False)
+        assert not result["pid_after_kill"]
+        # Nothing clears, moves, rewrites, or installs application data here.
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity", "--ez", "m03FixedIdentity", "true")
+        find("Item count: 2")
+        find("Local storage ready")
+        result["pid_after_restart"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_after_restart"] and result["pid_after_restart"] != result["pid_before"]
+        result["after_restart"] = database("after-restart")
+        assert result["after_restart"] == result["before_kill"]
+        assert_ui("after-restart-ui")
+        assert len(remote()["requests"]) == 9, "Cold startup must read SQLite without a network pull"
+        result["first_restart_complete_command_index"] = len(commands)
+        save()
+
+        # Only now create an independent second local DB. Preserve the first DB
+        # and its journal companions while the process is confirmed stopped.
+        adb("shell", "am", "force-stop", PACKAGE)
+        assert not adb("shell", "pidof", PACKAGE, check=False)
+        files = adb("shell", "run-as", PACKAGE, "ls", "files").splitlines()
+        for suffix in ("", "-wal", "-shm", "-journal"):
+            if "items.db" + suffix in files:
+                adb("shell", "run-as", PACKAGE, "mv", f"files/items.db{suffix}", f"files/m03-first.db{suffix}")
+        assert database("first-preserved", "m03-first.db") == result["before_kill"]
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        find("Item count: 0")
+        find("Local storage ready")
+        assert database("second-empty")["items"] == []
+        result["second"] = synchronize("second-final", FINAL, 10)
+        assert_ui("second-final-ui")
+        result["first_preserved"] = database("first-preserved-final", "m03-first.db")
+        assert result["first_preserved"] == result["before_kill"]
+        result["remote"] = remote()
+        assert result["remote"] == {"items": FINAL, "nextTimestamp": 1700000104000, "requests": expected_trace()}
+        (evidence / "remote-final.json").write_text(json.dumps(result["remote"], indent=2) + "\n")
+        assert os.getpid() == result["host_pid"]
+        result["status"] = "PASS"
+    except Exception as error:
+        result["status"], result["error"] = "FAIL", repr(error)
+    finally:
+        if "network_before" in result:
+            try:
+                result["network_after"] = network_state()
+                if result["network_after"] != result["network_before"]:
+                    result["status"], result["error"] = "FAIL", "Connectivity settings changed during M03"
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
+                result["status"], result["error"] = "FAIL", "Owned fixture needed forced cleanup"
+            if result["fixture_exit"] != 0:
+                result["status"], result["fixture_cleanup_error"] = "FAIL", "Owned fixture did not exit cleanly"
+        log.close()
+        result["adb_command_count"] = len(commands)
+        save()
+        print(json.dumps(result, indent=2), flush=True)
+    if result["status"] != "PASS":
+        raise AssertionError(result.get("error", "M03 did not complete"))
+
+
+if __name__ == "__main__":
+    main()
diff --git a/verification/M03.md b/verification/M03.md
new file mode 100644
index 0000000..fe6a20b
--- /dev/null
+++ b/verification/M03.md
@@ -0,0 +1,168 @@
+# M03 verification — attempt 1
+
+- Branch: `track/react-native`; START: `8baed88fe5e9a2c072c004c6dfe8e3dbadcba27f`.
+- Spec-Revision: `ed7baa246ee947c6778e0f84751c9b91cec7abfe`.
+- Frozen scenario SHA-256: `46589027d2fd585c5c7cc1ca86f409f0ec90c5eaba459c16f4843363cf4cd1e9`.
+- Raw evidence: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M03/`.
+- Android: the single API 34 Pixel 6 ARM64 emulator, `emulator-5554`, under an explicit exclusive lease.
+
+## Reproduction and implementation boundary
+
+The complete main README, all spec Markdown, protocol and frozen M03 scenario,
+then only this track's source/history were read. START and a clean tree were
+verified. No other track's implementation or records were inspected.
+
+Before production edits, the baseline added one test to the existing SQLite suite:
+two independent files start empty; creating Gamma at 1700000100000 in the first
+produces `item-001`, while the second remains empty, including after both files
+are closed/reopened. `baseline-jest-01` passed (one selected test, seven skipped).
+Its raw log contains both complete Item arrays. `baseline-01/source/` preserves
+the exact test, original production files and hashes. The captured M02 APK hash
+is `83b54f5a6f18ae5750f66702dbbc9ab579eef34c9e140ecd2071a532d59e4b68`.
+This is host file-backed SQLite via the existing library bridge adapter, not
+Android network evidence. The separate actual Android run is described below.
+
+The new fixture uses Node's HTTP server, fixed seeds, clock 1700000100000 with
+1000-ms accepted-mutation ticks, delay zero and no injected failure. Explicit
+foreground synchronization compares persisted local edits with the session's
+last shared snapshot, sends the required HTTP mutations, then atomically replaces
+SQLite rows. UI publication reads SQLite after commit. The schema remains v1.
+Production Item namespaces contain time/random nonces and the durable local
+counter; only tests inject `device`. Android's initial-prop override is debug-only.
+
+There is no durable pending intent, retry/idempotency/conflict policy, background
+work, push, state framework, custom native module, dependency addition or iOS code.
+The first explicit sync of a new session replaces local Items with canonical
+remote state; the UI warns about this. An unsent edit is not recoverably uploadable
+across session/process restart. These are staged limitations, not M05–M07 claims.
+
+## Exact rerun commands and invocation ledger
+
+Run npm with Node 22 on PATH and `/opt/homebrew/bin/npm`; Gradle additionally uses
+JDK17, the fixed SDK and the external React Native cache:
+
+```sh
+export PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH
+export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
+export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
+export GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-react-native
+/opt/homebrew/bin/npm run typecheck
+/opt/homebrew/bin/npm test -- --runInBand --watchman=false
+cd android
+./gradlew --no-daemon :app:assembleDebug :app:assembleDebugAndroidTest
+```
+
+Every named invocation has an immutable `.log` and `.command.json` with full argv,
+cwd, exit, timestamps and source hashes; `snapshots/<invocation>/` saves the source
+used (baseline source is under `baseline-01/`). No fixture acceptance was tuned.
+
+| Invocation | Command/result | Exit |
+|---|---|---:|
+| baseline-jest-01 | `npm test -- --runInBand --watchman=false __tests__/items.test.ts -t 'M03 baseline'`; expected isolation reproduced | 0 |
+| typecheck-01 | `npm run typecheck` | 0 |
+| jest-01 | Full suite; sandbox denied `listen 127.0.0.1:18081` with EPERM; 12 tests passed and three sync tests failed setup | 1 |
+| jest-02 | Same suite with approved local socket access; 15/15 passed | 0 |
+| build-01 | Both debug APKs; 99 Gradle tasks, 22 executed | 0 |
+| harness-syntax-01 | `python3 -m py_compile scripts/verify_m03.py` | 0 |
+| harness-syntax-02 | Same check after fixture-ownership and cleanup hardening, before Android execution | 0 |
+| android-clear-01 | `adb -s emulator-5554 shell pm clear com.mse.reactnative` before M01 regression | 0 |
+| android-ui-01 | Unchanged `M01ItemUiTest` on the leased emulator; 1 test, 0 failures/errors/skips; 96 seconds including Gradle | 0 |
+| typecheck-02 | Final TypeScript check | 0 |
+| jest-03 | Final full suite; 15/15 passed | 0 |
+| android-sync-01 | Full fixed M03 Android harness below; PASS in 128 seconds | 0 |
+
+The first socket denial also exposed an existing cleanup assumption: after a
+failed `beforeAll`, no SQLite test directory exists. A one-line guard prevents
+that secondary ENOENT error; the original failure output remains intact. One
+TRACK documentation patch missed its context and made no changes before the
+corrected patch (`edit-tool-failures.md`). Node's built-in SQLite experimental
+notice and the existing Hermes/Metro build warnings remain in raw logs.
+
+Host coverage asserts every method/path/body/status/response in the exact ten
+HTTP requests, all five fields after each sync, two independent database files,
+reopening both files, atomic rollback of one injected local INSERT failure, UI
+publication from a post-sync DB read, and distinct persisted production Item IDs.
+Supporting inputs were recorded before their first runs in
+`frozen-supporting-inputs.md` and `identity-supporting-inputs.md`.
+
+## Actual Android path
+
+The unchanged M01 test command is run from `android/` after the reset above:
+
+```sh
+ANDROID_SERIAL=emulator-5554 ./gradlew --no-daemon :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=com.mse.reactnative.M01ItemUiTest
+```
+
+From the branch root, with the device lease, no other listener on port 18081, and
+a fresh evidence directory, this single command owns the fixture lifecycle and
+performs the full M03 Android network/native-persistence verification:
+
+```sh
+python3 scripts/verify_m03.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M03/android-01
+```
+
+The harness records every adb invocation/exit and fixture control request, UI XML
+and screenshots, native SQLite files and integrity/schema/five-field inspections,
+the complete HTTP trace, external/app/fixture PIDs and fixture shutdown result.
+It never changes connectivity settings. The process restart occurs without data
+clear, reinstall or database changes. Only after all first-instance restart
+assertions finish does it stop the app, confirm no PID, preserve that DB and any
+WAL/SHM/journal companions under a distinct filename, and let the app create a
+fresh second SQLite file. Both files remain available for exact comparison.
+
+`android-ui-results-01/` contains actual JUnit XML and instrumentation output.
+The first and only M03 device invocation passed: host PID **87988** survived app
+PID **17540** being force-stopped, confirmed absent, then relaunched as **18474**.
+Commands 112–119 contain that boundary; there is no data reset, installation or
+file move between them. Native rows and the allocator were unchanged after the
+new process opened the database. The fixture still had exactly nine requests,
+proving the restarted UI did not fetch its displayed Items from the network.
+
+All first-instance restart assertions finished after command 127. Only then did
+commands 128–129 stop/confirm absence of the app, and 131–132 preserve `items.db`
+and its rollback journal as `m03-first.db` and its companion. There were no WAL/SHM
+files in this run. The restarted app created an empty independent `items.db`,
+whose first pull converged without altering the preserved first DB. The first
+allocator is 2 and the second is 1, consistent with their separate lifetimes.
+
+Both native database files and remote contain exactly:
+
+| id | title | completed | version | updatedAt |
+|---|---|---|---:|---:|
+| device-001 | Gamma | false | 1 | 1700000100000 |
+| remote-001 | Alpha synced | true | 3 | 1700000102000 |
+
+Beta/`remote-002` is absent. `remote-final.json` records the exact ten successful
+HTTP exchanges and next clock value 1700000104000, including the deletion tick.
+No request ordering is left to concurrent arrivals. `commands.json` contains
+159 adb commands: 157 exit 0 and two expected `pidof` exit-1 results after the two
+app force-stops. The actual RN fetch path reaches the host via `10.0.2.2`; the
+external harness uses only test-state HTTP reads, never Item mutation endpoints.
+
+The harness launched owned fixture PID **87999** using the `fixture_command`
+array in `result.json`; it sent SIGTERM through `Popen.terminate()` and waited for
+**exit 0**. It made no connectivity changes: airplane/Wi-Fi/mobile-data settings
+were **0/1/1** before and after. The device lease was released immediately after
+this completed run. No additional device action is performed after release.
+
+| Artifact | SHA-256 |
+|---|---|
+| Tested app APK | `e668de3a3954db9c40b795f60a8316001cb63f3f0aca4b498c20ab57e60f7561` |
+| Unchanged M01 test APK | `7259ba83ec046d34a3e489ebec46ddcf9ce89dd136a934784c0c89669795b7c1` |
+| Executed final M03 harness | `b81eb511088a02d1ebcaddc7310617b8f65a653c35e905a6e5e96850743a22b9` |
+| Executed fixture | `003b246c5d89c739e9d618322fa1a6461eec631177455c175ddf9223b25d360f` |
+| First DB before kill, after restart and preserved final | `ecbad678db5f43efdabe865cf719046632dcbb1cfe48303183e7c11ebca83ce7` |
+| Independent second final DB | `f201864456daf66ca5dca46ce270004d4a614f8f755cb3148718faac3ce257a3` |
+
+The post-run artifact audit exited 0 (`artifact-audit-01.json`): all eight changed
+or relevant production files are byte-identical to those in the successful build,
+the executed harness is unchanged, APK hashes match, every native final artifact
+has the exact table, and all file moves follow the completed restart boundary.
+
+Totals: one pre-fix baseline Jest invocation; three full Jest invocations (first
+blocked by sandbox sockets, final two 15/15 passed); two successful typechecks;
+one explicit APK build; one successful unchanged Android UI invocation; two
+successful harness syntax checks; one full successful M03 Android invocation.
+No fixture/input tuning, performance benchmark, battery soak, alternate Android
+profile or later Thread work occurred. Main's independent verification remains
+required before a progress tag; no tag or main index entry is created here.
