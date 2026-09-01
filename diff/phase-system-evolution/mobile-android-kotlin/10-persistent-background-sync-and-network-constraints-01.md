# M10 — Persistent Background Sync와 Network Constraints

## `test(M10): preserve accepted foreground-only baseline`

diff --git a/TRACK.md b/TRACK.md
index 78dd05f..a46252a 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -243,6 +243,17 @@ The fixture12 baseline results and test APK were reused unchanged. Main confirme
 frozen execution bytes and actual cleanup. Attempt1, repair0/2; only this reporting and
 `verification/M09.md` changed after runtime. Final history/tag audit remains main-owned.
 
+## M10 boundary — baseline accepted
+
+Main accepted the unchanged M09 app's foreground-only limitation after two preserved
+harness failures. The repaired baseline used an explicitly labelled native preseed, then
+ordinary offline startup, HOME and same-UID SIGKILL. The original queued identity remained
+durable while the package stayed unstopped and absent through ten seconds online, with
+no WorkManager database, registered job or application HTTP request. This is not production
+creation or scheduler acceptance. `verification/M10.md` retains A1/A2 failures, the accepted
+A3 evidence and the user's subsequent repair/verification authorization. M10 implementation
+and both final OS cases remain outstanding; no M10 completion or tag is claimed.
+
 ## Pinned build
 
 - JDK 17, Gradle 8.7 (wrapper distribution SHA-256 pinned)
diff --git a/verification/M10-inputs.json b/verification/M10-inputs.json
new file mode 100644
index 0000000..abf14b9
--- /dev/null
+++ b/verification/M10-inputs.json
@@ -0,0 +1,52 @@
+{
+  "thread": "M10",
+  "profile": "phase-1",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "itemId": "work-001",
+  "title": "Background item",
+  "completed": false,
+  "clientMutationId": "m10-create-001",
+  "payloadHash": "b096e8b4ea45527ced6766b4aaf04fe9309e73b19437d84aa73bd2ccadb17359",
+  "canonicalRequest": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"work-001\",\"title\":\"Background item\"}}",
+  "localTimestamp": 1700000700000,
+  "firstSuccessTimestamp": 1700000700000,
+  "retryCeiling": 3,
+  "backoffMs": [10000, 20000],
+  "cases": [
+    {"name": "A", "statuses": [503, 503, 201], "applied": 1, "pending": 0},
+    {"name": "B", "statuses": [503, 503, 503], "applied": 0, "pending": 1, "noFourthAutomaticRequest": true}
+  ],
+  "baseline": {
+    "case": "A",
+    "setup": "Exact schema5 native SQLite preseed before ordinary M09 Activity launch; not production creation or scheduler registration proof.",
+    "app": "unchanged verified M09",
+    "testApkUsed": false,
+    "initialLaunchOffline": true,
+    "expectedInitialStatus": "Stale local data · sync error",
+    "expectedDispatched": 1,
+    "lossMethod": "HOME then same-UID run-as kill -9 of the verified PID",
+    "packageStoppedAfterLoss": false,
+    "activityStartsAfterLoss": 0,
+    "instrumentationRuns": 0,
+    "jobs": 0,
+    "workDatabases": 0,
+    "onlineObservationSeconds": 10,
+    "expectedHttpRequests": 0,
+    "expectedPending": 1,
+    "fixtureReset": [
+      {"path": "/__m09/reset", "body": {"case": "A"}},
+      {"path": "/__control", "body": {"nextTimestamp": 1700000700000}}
+    ]
+  },
+  "harness": {
+    "schemaVersion": 5,
+    "uiWaitSeconds": 30,
+    "networkWaitSeconds": 30,
+    "adbTimeoutSeconds": 45,
+    "preSeedTeardownWaitSeconds": 30,
+    "lossWaitSeconds": 15
+  },
+  "budgetOwner": "root",
+  "maximumFullExecutionsPerCase": 3,
+  "repairLimit": 2
+}
diff --git a/verification/M10.md b/verification/M10.md
new file mode 100644
index 0000000..b2b42ca
--- /dev/null
+++ b/verification/M10.md
@@ -0,0 +1,54 @@
+# M10 verification ledger — baseline accepted
+
+- Profile: `phase-1`; Thread: `M10`; branch: `track/android-kotlin`.
+- SPEC_REVISION: `61280dd86ce88b6e431f408241c0998a275960aa`.
+- START: `6c3b11fb9aa6ecdcfea1312e083183f83ef524d5` (verified M09).
+- Current attempt3 / cumulative repair2. Actual fixed-case usage A3 / B0, never reset.
+- Status: unchanged-M09 limitation accepted by root; M10 product and final A/B acceptance outstanding.
+
+Evidence root `E`:
+`/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M10/`.
+Each run keeps its exact argv, exits, timestamps, raw outputs, hashes and cleanup.
+
+## Preserved attempts
+
+| Run | Actual result | Evidence under E |
+| --- | --- | --- |
+| Initial support | Host5 PASS; unchanged M09 app/test APKs reused; no build or fixture-suite repeat | `baseline-host01.*`, `baseline-frozen-01/manifest.json` |
+| Baseline A1 | FAIL before Activity launch: cmd022 exec-in exited0 without echo; 30 adb commands,1.828s | `baseline-android-A1/` |
+| Repair1 | Seed transport only: raw shell-v2 remote completion and exact native readback; focused host3 PASS | `repair1-host01.*`, `repair1-frozen-01/manifest.json` |
+| Baseline A2 | Seed write/readback45056 bytes PASS; FAIL at cmd036 because unrelated queryability User0 was counted.44 adb commands,13.710s; no HOME/SIGKILL | `repair1-baseline-android-A2/` |
+| Repair2 | Target-package User0 parser only; actual raw dump plus malformed/duplicate/missing-state coverage, host3 PASS | `repair2-host01.*`, `repair2-frozen-01/manifest.json` |
+| Baseline A3 | LIMITATION_REPRODUCED;83 adb commands,24.387s; root audited193 raw artifacts and4 native DB/WAL snapshots | `repair2-baseline-android-A3/` |
+
+A1 and A2 remain charged and failed. The earlier budget stops are retained, not erased.
+The user's [repair authorization](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/REPAIR-AUTHORIZATION-2026-08-29.md)
+permits necessary bounded repairs and verification for fixable issues while preserving all
+counts and the product's max3 HTTP ceiling. [Repair2 dispatch](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/resume-04/repair2-dispatch.json)
+discloses existing-owner fallback after both fresh and independent-agent activation failed.
+The initial ADB source-download sandbox DNS failure and subsequent approved HTTP200 fetch
+also remain under `repair1-adb-source-01/` and `repair1-adb-source-02/`.
+
+## Accepted limitation, not final scheduler proof
+
+The test-only native seed is schema5 with work-001 / Background item / false, queued identity
+m10-create-001 and its frozen payload hash. This is baseline setup, not production creation.
+Ordinary offline M09 startup marks the same intent dispatched and presents its sync error.
+After HOME, PID4379/UID10195 was verified and killed once by the same UID. No app entrypoint
+followed. The app stayed absent with stopped=false, no registered job or WorkManager database,
+and no application HTTP through10.010s unattended online. All four native snapshots retain
+the original intent, Item and allocator1; pending remains1.
+
+Root independently checked native SQLite/WAL, PID/UID, UI, request logs and frozen bytes in
+[the A3 audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M10/main-repair2-baseline-audit.json).
+[Cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/android-kotlin/M10/main-repair2-cleanup.json)
+confirms owned fixture29652 exited0, PID absent/port18080 free, app absent and network0/1/1.
+
+Frozen support: `repair2-frozen-01/manifest.json`, SHA256
+`6b247932306f5eeec24a82373f01b9b73448d5c1526e2ac304cf832094c10ad9`.
+All54 START files were unchanged during baseline execution; only two verification support
+files were added. The same app `3f4b3fc39b1d9d6906fb162aa608e770debbe05f74f0de06b3aaded17d1ef909`,
+test reference `16a5e3967d68e6790431e831f08ded7b1294172174b64a3c979d6883390a35b0`,
+and seed `58325cf3cadaa82af1f5d31e8623d0a1dc7d5eecb7dfeee3b34e561f99122e00` were reused.
+No owner device run, product build or final OS test is claimed. Root separately
+authorized minimal M10 implementation after accepting this baseline.
diff --git a/verification/background_baseline.py b/verification/background_baseline.py
new file mode 100644
index 0000000..0fd5bc1
--- /dev/null
+++ b/verification/background_baseline.py
@@ -0,0 +1,329 @@
+#!/usr/bin/env python3
+"""M10-A limitation only, using the unchanged verified M09 app and a baseline preseed.
+
+Root owns the capped device invocation. No production creation/registration is
+claimed for the preseed, and no target entrypoint is invoked after actual SIGKILL.
+"""
+import argparse
+import hashlib
+import json
+import os
+from pathlib import Path
+import re
+import shlex
+import sqlite3
+import subprocess
+import time
+
+from offline_queue_restart import OfflineQueueScenario
+from process_restart import PACKAGE, SERIAL
+from startup_recovery import StartupRecoveryScenario
+from version_conflict_restart import VersionConflictScenario
+
+
+INPUTS_PATH = Path(__file__).with_name('M10-inputs.json')
+INPUTS = json.loads(INPUTS_PATH.read_text())
+SCHEMA_PATH = Path(__file__).parents[1] / 'app/schemas/com.mobilesystemsevolution.kotlin.ItemDatabase/5.json'
+
+
+def create_baseline_seed(path):
+    """Host setup only; the fixed case later requires real product edit/registration."""
+    assert not path.exists()
+    schema = json.loads(SCHEMA_PATH.read_text())['database']
+    with sqlite3.connect(path) as database:
+        for entity in schema['entities']:
+            database.execute(entity['createSql'].replace('${TABLE_NAME}', entity['tableName']))
+        for statement in schema['setupQueries']:
+            database.execute(statement)
+        database.execute('PRAGMA user_version=5')
+        database.execute('INSERT INTO items VALUES(?,?,?,?,?)',
+                         (INPUTS['itemId'], INPUTS['title'], 0, 0, INPUTS['localTimestamp']))
+        database.execute('INSERT INTO pending_mutations VALUES(?,?,?,?,?,?,?,?,?,?)',
+                         (1, INPUTS['itemId'], 'CREATE', INPUTS['title'], 0,
+                          INPUTS['clientMutationId'], INPUTS['payloadHash'], None, None, 0))
+
+
+def registered_jobs(text):
+    """The API34 header is a GLOBAL count even in a package-filtered dump."""
+    lines = text.splitlines()
+    headers = [index for index, line in enumerate(lines) if re.fullmatch(r'\s*Registered \d+ jobs:', line)]
+    assert len(headers) == 1, 'Malformed package-filtered JobScheduler dump'
+    start = headers[0]
+    indent = len(lines[start]) - len(lines[start].lstrip())
+    block = []
+    for line in lines[start + 1:]:
+        if line.strip() and len(line) - len(line.lstrip()) <= indent:
+            break
+        if line.strip():
+            block.append(line.strip())
+    assert block, 'Missing registered-job body'
+    if block == ['None.']:
+        return []
+    jobs = [line for line in block if line.startswith('JOB ')]
+    assert jobs and 'None.' not in block, 'Malformed registered-job body'
+    return jobs
+
+
+def package_stopped(text):
+    def children(lines, start):
+        indent = len(lines[start]) - len(lines[start].lstrip())
+        block = []
+        for line in lines[start + 1:]:
+            if line.strip() and len(line) - len(line.lstrip()) <= indent:
+                break
+            block.append(line)
+        return block
+
+    lines = text.splitlines()
+    sections = [index for index, line in enumerate(lines) if line.strip() == 'Packages:']
+    assert len(sections) == 1, 'Missing or ambiguous Packages section'
+    packages = children(lines, sections[0])
+    header = rf'\s+Package \[{re.escape(PACKAGE)}\] \([^)]+\):'
+    matches = [index for index, line in enumerate(packages) if re.fullmatch(header, line)]
+    assert len(matches) == 1, 'Missing or ambiguous target package block'
+    users = [line for line in children(packages, matches[0]) if re.match(r'\s*User 0:', line)]
+    assert len(users) == 1, 'Missing or ambiguous user0 package state'
+    states = re.findall(r'(?<!\S)stopped=(\S*)', users[0])
+    assert len(states) == 1 and states[0] in ('true', 'false'), 'Missing, duplicate or malformed stopped flag'
+    return states[0] == 'true'
+
+
+def resumed_activity_lines(text):
+    return [line.strip() for line in text.splitlines()
+            if re.match(r'\s*(?:topResumedActivity=|mResumedActivity:)', line)
+            and 'ActivityRecord{' in line]
+
+
+class BackgroundBaseline(OfflineQueueScenario):
+    pre_seed_teardown = VersionConflictScenario.pre_seed_teardown
+    capture = StartupRecoveryScenario.capture
+
+    def __init__(self, args):
+        super().__init__(args)
+        self.invocations = []
+        self.result.update(scenario='M10', case='A', expectation='foreground-only-limitation',
+                           harnessSha256=hashlib.sha256(Path(__file__).read_bytes()).hexdigest(),
+                           inputsSha256=hashlib.sha256(INPUTS_PATH.read_bytes()).hexdigest(),
+                           seedSha256=hashlib.sha256(args.seed_db.read_bytes()).hexdigest(),
+                           baselineSetup=INPUTS['baseline']['setup'], testApkUsed=False,
+                           productionCreationProved=False, schedulerRegistrationProved=False,
+                           fixtureLog=str(args.fixture_log.resolve()))
+
+    def adb(self, *args, **kwargs):
+        number, started = self.command_count + 1, time.monotonic()
+        self.invocations.append(args)
+        try:
+            return super().adb(*args, **kwargs)
+        finally:
+            with (self.output / 'command-times.jsonl').open('a') as log:
+                log.write(json.dumps(dict(command=number, arguments=args,
+                                          startedMonotonic=started, endedMonotonic=time.monotonic())) + '\n')
+
+    def install_seed(self):
+        self.adb('shell', 'run-as', PACKAGE, 'mkdir', '-p', 'databases')
+        data = self.args.seed_db.read_bytes()
+        assert hashlib.sha256(data).hexdigest() == self.result['seedSha256']
+        transfer = self.result['seedTransfer'] = dict(seedSha256=self.result['seedSha256'],
+                                                     seedBytes=len(data), commands=[])
+        # Unlike exec-in, shell -T requires shell-v2 and waits for the remote exit
+        # after sending stdin EOF. No PTY or remote-shell redirection changes bytes.
+        steps = (
+            ('write', ('shell', '-T', 'run-as', PACKAGE, 'tee', 'databases/items.db'), data),
+            ('readback', ('shell', '-T', '-n', 'run-as', PACKAGE, 'cat', 'databases/items.db'), None),
+        )
+        for phase, arguments, stdin in steps:
+            command = [str(self.args.adb), '-s', SERIAL, *arguments]
+            self.command_count += 1
+            self.invocations.append(arguments)
+            prefix = self.output / f'command-{self.command_count:03d}'
+            if stdin is not None:
+                prefix.with_suffix('.stdin').write_bytes(stdin)
+            started = time.monotonic()
+            with (self.output / 'commands.txt').open('a') as ledger:
+                ledger.write(f'{self.command_count:03d} {shlex.join(command)}\n')
+            result = subprocess.run(command, input=stdin, capture_output=True,
+                                    timeout=INPUTS['harness']['adbTimeoutSeconds'])
+            prefix.with_suffix('.stdout').write_bytes(result.stdout)
+            prefix.with_suffix('.stderr').write_bytes(result.stderr)
+            with (self.output / 'commands.txt').open('a') as ledger:
+                ledger.write(f'    exit={result.returncode}\n')
+            record = dict(phase=phase, command=self.command_count, arguments=arguments,
+                          startedMonotonic=started, endedMonotonic=time.monotonic(),
+                          exit=result.returncode, stdinBytes=len(stdin) if stdin is not None else 0,
+                          stdinSha256=self.result['seedSha256'] if stdin is not None else None,
+                          stdoutBytes=len(result.stdout), stdoutSha256=hashlib.sha256(result.stdout).hexdigest(),
+                          stderrBytes=len(result.stderr), stderrSha256=hashlib.sha256(result.stderr).hexdigest(),
+                          exactSeedBytes=result.stdout == data)
+            with (self.output / 'command-times.jsonl').open('a') as log:
+                log.write(json.dumps(record) + '\n')
+            transfer['commands'].append(record)
+            (self.output / 'seed-transfer.json').write_text(json.dumps(transfer, indent=2) + '\n')
+            assert result.returncode == 0 and result.stderr == b'', f'Native baseline seed {phase} failed'
+            assert result.stdout == data, f'Native baseline seed {phase} bytes differ'
+
+    def assert_pending(self, captured, dispatched):
+        assert captured == dict(
+            items=[dict(id=INPUTS['itemId'], title=INPUTS['title'], completed=0, version=0,
+                        updatedAt=INPUTS['localTimestamp'])],
+            pending=[dict(sequence=1, itemId=INPUTS['itemId'], operation='CREATE', title=INPUTS['title'],
+                          completed=0, clientMutationId=INPUTS['clientMutationId'], payloadHash=INPUTS['payloadHash'],
+                          terminalError=None, baseVersion=None, dispatched=int(dispatched))],
+            acknowledged=[], tombstones=[], refreshMetadata=[], allocator=[['pending_mutations', 1]],
+        ), captured
+
+    def scheduler_state(self, stage):
+        stopped = package_stopped(self.adb('shell', 'dumpsys', 'package', PACKAGE))
+        jobs = registered_jobs(self.adb('shell', 'dumpsys', 'jobscheduler', PACKAGE))
+        files = self.adb('shell', 'run-as', PACKAGE, 'find', '.', '-type', 'f').splitlines()
+        work_databases = [name for name in files if Path(name).name.startswith('androidx.work.workdb')]
+        state = dict(packageStopped=stopped, jobs=jobs, workDatabaseFiles=work_databases)
+        assert not stopped and not jobs and not work_databases, state
+        (self.output / f'{stage}-scheduler.json').write_text(json.dumps(state, indent=2) + '\n')
+        self.result.setdefault('schedulerObservations', {})[stage] = state
+        return state
+
+    def assert_no_http(self):
+        state = self.http('/__m09')
+        assert state['items'] == [] and state['nextTimestamp'] == INPUTS['firstSuccessTimestamp'], state
+        assert (state['applied'], state['duplicates'], state['mutationRequests'], state['droppedResponses']) == (0, 0, 0, 0), state
+        control = self.http('/__control')
+        assert control['getRequests'] == 0, control
+        return state
+
+    def wait_background(self):
+        deadline = time.monotonic() + INPUTS['harness']['uiWaitSeconds']
+        while True:
+            resumed = resumed_activity_lines(self.adb('shell', 'dumpsys', 'activity', 'activities'))
+            if resumed and all(PACKAGE not in line for line in resumed):
+                self.result['backgroundResumedActivities'] = resumed
+                return
+            if time.monotonic() >= deadline:
+                raise AssertionError(f'App did not leave resumed activities: {resumed}')
+            time.sleep(0.2)
+
+    def wait_absent(self):
+        deadline = time.monotonic() + INPUTS['harness']['lossWaitSeconds']
+        while True:
+            pid = self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+            if not pid:
+                return time.monotonic()
+            if time.monotonic() >= deadline:
+                raise AssertionError(f'App PID remained after the single SIGKILL: {pid}')
+            time.sleep(0.2)
+
+    def run(self):
+        self.result['initialNetwork'] = self.wait_network(True)
+        self.adb('shell', 'am', 'force-stop', PACKAGE)
+        self.pre_seed_teardown('before-install')
+        self.adb('install', '-r', str(self.args.apk.resolve()))
+        assert self.adb('shell', 'pm', 'clear', PACKAGE) == 'Success'
+        self.pre_seed_teardown('after-clear')
+        self.adb('shell', 'input', 'keyevent', 'KEYCODE_WAKEUP')
+        self.adb('shell', 'wm', 'dismiss-keyguard')
+        self.adb('logcat', '-c')
+        for control in INPUTS['baseline']['fixtureReset']:
+            self.http(control['path'], control['body'])
+        self.fixture_offset = self.args.fixture_log.stat().st_size
+        self.result['fixtureLogStartOffset'] = self.fixture_offset
+        self.assert_no_http()
+        self.go_offline()
+        self.install_seed()
+        self.assert_pending(self.capture('preseed'), dispatched=False)
+
+        self.adb('shell', 'am', 'start', '-W', '-n', f'{PACKAGE}/.MainActivity')
+        self.completed_ui([INPUTS['title']])
+        self.wait_text(INPUTS['baseline']['expectedInitialStatus'])
+        self.wait_text('Pending changes: 1')
+        queued = self.capture('foreground-offline')
+        self.assert_pending(queued, dispatched=True)
+        self.assert_no_http()
+        self.scheduler_state('foreground-offline')
+        self.adb('shell', 'input', 'keyevent', 'KEYCODE_HOME')
+        self.wait_background()
+        uid = self.adb('shell', 'run-as', PACKAGE, 'id', '-u')
+        pid = self.adb('shell', 'pidof', PACKAGE)
+        assert uid.isdigit() and pid.isdigit() and int(pid) > 1
+        process = self.adb('shell', 'run-as', PACKAGE, 'cat', f'/proc/{pid}/status')
+        actual_uid = re.search(r'^Uid:\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s*$', process, re.MULTILINE)
+        assert actual_uid and all(value == uid for value in actual_uid.groups()), 'SIGKILL target UID mismatch'
+        assert self.adb('shell', 'pidof', PACKAGE) == pid
+        self.result.update(pidBefore=int(pid), targetUid=int(uid),
+                           lossMethod='same-UID SIGKILL after HOME', lossCommand=self.command_count + 1,
+                           killRequestedMonotonic=time.monotonic())
+        self.adb('shell', 'run-as', PACKAGE, 'kill', '-9', pid)
+        self.result['absenceMonotonic'] = self.wait_absent()
+        killed = self.capture('killed-offline')
+        assert killed == queued
+        self.scheduler_state('killed-offline')
+        self.result['offlineAfterLoss'] = self.wait_network(False)
+        self.assert_no_http()
+
+        self.result['onlineBeforeObservation'] = self.restore_network()
+        assert not self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        start = time.monotonic()
+        time.sleep(INPUTS['baseline']['onlineObservationSeconds'])
+        end = time.monotonic()
+        assert not self.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+        self.result['onlineAfterObservation'] = self.wait_network(True)
+        self.result['onlineObservation'] = dict(startedMonotonic=start, endedMonotonic=end, seconds=end - start)
+        assert end - start >= INPUTS['baseline']['onlineObservationSeconds']
+        final = self.capture('online-unattended')
+        assert final == queued
+        self.scheduler_state('online-unattended')
+        self.result['finalRemote'] = self.assert_no_http()
+        self.adb('logcat', '-d', '-v', 'threadtime')
+        after_loss = self.invocations[self.result['lossCommand']:]
+        forbidden = [args for args in after_loss if args[:3] in (
+            ('shell', 'am', 'start'), ('shell', 'am', 'instrument'), ('shell', 'am', 'force-stop'),
+            ('shell', 'pm', 'clear'), ('exec-in', 'run-as', PACKAGE),
+        ) or args[:1] == ('install',)]
+        assert forbidden == [], forbidden
+        fixture = self.args.fixture_log.read_bytes()
+        events = [json.loads(line) for line in fixture[self.fixture_offset:].decode().splitlines() if line.startswith('{')]
+        app_requests = [event for event in events if event.get('path') == '/items' or event.get('path', '').startswith('/items/')]
+        assert app_requests == [], app_requests
+        self.result.update(status='LIMITATION_REPRODUCED', pending=1, actualHttpRequests=0,
+                           nativeSnapshots=4, fixtureLogEndOffset=len(fixture),
+                           activityStartsAfterLoss=0, instrumentationRuns=0, automaticJobs=0,
+                           automaticWorkDatabases=0, packageStoppedBeforeCleanup=False,
+                           pidAfterObservation=None, noAppEntrypointAfterLoss=True)
+
+
+def main():
+    parser = argparse.ArgumentParser(description=__doc__)
+    parser.add_argument('--expect', choices=['limitation'], required=True)
+    parser.add_argument('--apk', type=Path, required=True)
+    parser.add_argument('--test-apk', type=Path, required=True, help='Hash reference only; never installed or invoked')
+    parser.add_argument('--seed-db', type=Path, required=True)
+    parser.add_argument('--schema-version', type=int, choices=[5], required=True)
+    parser.add_argument('--adb', type=Path, required=True)
+    parser.add_argument('--fixture-log', type=Path, required=True)
+    parser.add_argument('--output', type=Path, required=True)
+    args = parser.parse_args()
+    scenario = BackgroundBaseline(args)
+    started = time.monotonic()
+    try:
+        scenario.run()
+    except Exception as error:
+        scenario.result.update(status='FAIL', error=repr(error))
+        raise
+    finally:
+        try:
+            # Outside the observed loss/online window; never used as the M10 boundary.
+            scenario.adb('shell', 'am', 'force-stop', PACKAGE)
+            absent = not scenario.adb('shell', 'pidof', PACKAGE, allow_failure=True)
+            network = scenario.restore_network()
+            scenario.result['cleanup'] = dict(appAbsent=absent, network=network)
+            assert absent
+        except Exception as cleanup_error:
+            scenario.result.update(status='FAIL', cleanupError=repr(cleanup_error))
+            raise
+        finally:
+            scenario.result.update(adbCommands=scenario.command_count,
+                                   elapsedSeconds=round(time.monotonic() - started, 3))
+            (scenario.output / 'result.json').write_text(json.dumps(scenario.result, indent=2) + '\n')
+            print(json.dumps(scenario.result, indent=2), flush=True)
+
+
+if __name__ == '__main__':
+    main()


