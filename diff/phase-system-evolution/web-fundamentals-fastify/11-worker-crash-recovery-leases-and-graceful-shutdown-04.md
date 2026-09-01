## `test: preserve E11 recovery proof and repair provenance`

diff --git a/.github/workflows/check.yml b/.github/workflows/check.yml
index a421129..1bc9972 100644
--- a/.github/workflows/check.yml
+++ b/.github/workflows/check.yml
@@ -35,6 +35,8 @@ jobs:
         run: npm run test:functional
       - name: Request identity and competing workers
         run: npm run test:ownership
+      - name: Worker crash recovery and shutdown
+        run: npm run test:recovery
       - name: Install pinned Chromium
         run: npx playwright install --with-deps chromium
       - name: Production build and browser E2E
diff --git a/TRACK.md b/TRACK.md
index 6127dcd..d1fbe88 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -617,3 +617,47 @@ worker ID is refused completion. Production Chromium additionally holds a real
 202 after commit, loses that delivery, then verifies manual retransmission and
 a new acknowledged intention. The frozen files and prior evidence are unchanged;
 live regression helpers supply a fresh key for each prior intentional Check POST.
+
+## E11 lease recovery and bounded shutdown
+
+Migration `008_check_lease.sql` gives each RUNNING CheckRun a durable five-second
+lease, including pre-upgrade RUNNING rows. Stop old API/worker processes before
+applying it. Claim stores the owner, start time and expiry in one transaction.
+There is no lease renewal, requeue or automatic retry; the HTTP deadline remains
+two seconds.
+
+Completion uses one connection for `BEGIN`, the owner/RUNNING/live-lease guarded
+UPDATE, a lease recheck after any row-lock wait, and `COMMIT`. A worker killed
+before that commit cannot leave an autocommit result pending behind the lock.
+The worker loop calls `recoverExpiredChecks` with the database clock: expired
+RUNNING rows become ABORTED with the same ID and null HTTP status, latency and
+failure reason. Recovery performs no endpoint request. Already-terminal rows
+remain unchanged. The API returns ABORTED in terminal history and the browser
+shows its unavailable endpoint fields as **Unknown**. Existing history filters
+remain SUCCEEDED/FAILED; the unfiltered history includes ABORTED.
+
+Reusing the original manual key returns its original run and state. A new valid
+key creates a new QUEUED run without reopening the old terminal row.
+
+On SIGTERM the worker acknowledges stopping, stops new claims and drains only
+its current check. A three-second timer covers the drain and database close;
+normal completion exits zero, while a stuck shutdown exits nonzero and leaves
+uncommitted uncertainty for lease recovery. The fixed process test exercises
+the normal drain with one held response and an unchanged second QUEUED run.
+The stuck-deadline branch was inspected in source, not separately executed.
+
+`fnm exec --using 24.19.0 npm run test:recovery` runs the four frozen SIGKILL
+checkpoints and the SIGTERM case explicitly; it is outside default `npm test`.
+It owns only the five `e11_*` schemas named in `evidence/phase-1/E11/scenario.json`,
+refuses occupied ports 4311–4314 and cleans its processes and schemas. Exact
+expiry is observed through a separate Node test process calling the production
+recovery operation at lease minus 1ms and equality. This adapter does not claim
+to be a continuously running CLI worker; the actual CLI loop uses that same
+operation with the database clock.
+
+E11 evidence preserves both baseline invocations, the original terminal-write
+failure, and the original author's targeted passing correction. The second
+bounded repair adopted that source unchanged and recorded its provenance in
+`evidence/phase-1/E11/verification-attempt3.json`. The corrected full-suite and
+regression acceptance remain for root's final verification; no additional
+baseline or recovery run was spent by that repair.
diff --git a/evidence/phase-1/E11/author-before_commit-recheck.json b/evidence/phase-1/E11/author-before_commit-recheck.json
new file mode 100644
index 0000000..898ea20
--- /dev/null
+++ b/evidence/phase-1/E11/author-before_commit-recheck.json
@@ -0,0 +1,90 @@
+{
+  "result": "PASS",
+  "checkpoint": {
+    "name": "before_commit",
+    "schema": "e11_before_commit",
+    "barrier": "hold CheckRun row lock; release HTTP200; worker terminal UPDATE waiting on that lock",
+    "outboundCalls": 1,
+    "recoveredState": "ABORTED"
+  },
+  "observerPid": 84709,
+  "workerPid": 84711,
+  "workerId": "c4091677-4284-4d7c-9980-1730e36c94d2",
+  "exit": {
+    "code": null,
+    "signal": "SIGKILL",
+    "forced": false
+  },
+  "before": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "RUNNING",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:01:35.019Z",
+    "finished_at": null,
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "c4091677-4284-4d7c-9980-1730e36c94d2",
+    "lease_expires_at": "2026-08-28T05:01:40.019Z"
+  },
+  "recovered": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:01:35.019Z",
+    "finished_at": "2026-08-28T05:01:40.019Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "c4091677-4284-4d7c-9980-1730e36c94d2",
+    "lease_expires_at": "2026-08-28T05:01:40.019Z"
+  },
+  "beforeExpiry": {
+    "pid": 84713,
+    "clock": "2026-08-28T05:01:40.018Z",
+    "recovered": []
+  },
+  "atExpiry": {
+    "pid": 84714,
+    "clock": "2026-08-28T05:01:40.019Z",
+    "recovered": [
+      {
+        "id": "11000000-0000-4000-a000-000000000001",
+        "monitorId": "11000000-0000-4000-9000-000000000001",
+        "trigger": "MANUAL",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": null,
+        "startedAt": "2026-08-28T05:01:35.019Z",
+        "finishedAt": "2026-08-28T05:01:40.019Z"
+      }
+    ]
+  },
+  "staleCompletionWrites": 0,
+  "originalIdentityReplay": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitorId": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "httpStatus": null,
+    "latencyMs": null,
+    "failureReason": null,
+    "startedAt": "2026-08-28T05:01:35.019Z",
+    "finishedAt": "2026-08-28T05:01:40.019Z"
+  },
+  "retryId": null,
+  "outboundCalls": 1,
+  "recoveryProcess": "separate Node process invoking the production recoverExpiredChecks operation; the real CLI worker loop invokes the same operation",
+  "durationMs": 1001
+}
diff --git a/evidence/phase-1/E11/author-recovery-1-after_commit.json b/evidence/phase-1/E11/author-recovery-1-after_commit.json
new file mode 100644
index 0000000..a97c472
--- /dev/null
+++ b/evidence/phase-1/E11/author-recovery-1-after_commit.json
@@ -0,0 +1,78 @@
+{
+  "result": "PASS",
+  "checkpoint": {
+    "name": "after_commit",
+    "schema": "e11_after_commit",
+    "barrier": "HTTP200 terminal row observed through a separate connection after COMMIT",
+    "outboundCalls": 1,
+    "recoveredState": "SUCCEEDED"
+  },
+  "observerPid": 84207,
+  "workerPid": 84218,
+  "workerId": "6c2d9fb3-b4dd-4d18-bb7c-f2cdcd5fe0ce",
+  "exit": {
+    "code": null,
+    "signal": "SIGKILL",
+    "forced": false
+  },
+  "before": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "http_status": 200,
+    "latency_ms": 3,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:23.680Z",
+    "finished_at": "2026-08-28T05:00:23.687Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "6c2d9fb3-b4dd-4d18-bb7c-f2cdcd5fe0ce",
+    "lease_expires_at": "2026-08-28T05:00:28.680Z"
+  },
+  "recovered": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "http_status": 200,
+    "latency_ms": 3,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:23.680Z",
+    "finished_at": "2026-08-28T05:00:23.687Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "6c2d9fb3-b4dd-4d18-bb7c-f2cdcd5fe0ce",
+    "lease_expires_at": "2026-08-28T05:00:28.680Z"
+  },
+  "beforeExpiry": {
+    "pid": 84219,
+    "clock": "2026-08-28T05:00:28.679Z",
+    "recovered": []
+  },
+  "atExpiry": {
+    "pid": 84220,
+    "clock": "2026-08-28T05:00:28.680Z",
+    "recovered": []
+  },
+  "staleCompletionWrites": 0,
+  "originalIdentityReplay": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitorId": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "httpStatus": 200,
+    "latencyMs": 3,
+    "failureReason": null,
+    "startedAt": "2026-08-28T05:00:23.680Z",
+    "finishedAt": "2026-08-28T05:00:23.687Z"
+  },
+  "retryId": null,
+  "outboundCalls": 1,
+  "recoveryProcess": "separate Node process invoking the production recoverExpiredChecks operation; the real CLI worker loop invokes the same operation",
+  "durationMs": 929
+}
diff --git a/evidence/phase-1/E11/author-recovery-1-before_io.json b/evidence/phase-1/E11/author-recovery-1-before_io.json
new file mode 100644
index 0000000..675a45f
--- /dev/null
+++ b/evidence/phase-1/E11/author-recovery-1-before_io.json
@@ -0,0 +1,90 @@
+{
+  "result": "PASS",
+  "checkpoint": {
+    "name": "before_io",
+    "schema": "e11_before_io",
+    "barrier": "monitors ACCESS EXCLUSIVE; committed RUNNING and worker Monitor SELECT waiting on that lock",
+    "outboundCalls": 0,
+    "recoveredState": "ABORTED"
+  },
+  "observerPid": 84207,
+  "workerPid": 84208,
+  "workerId": "6a1d383e-e5f5-47f0-b648-984b40d32710",
+  "exit": {
+    "code": null,
+    "signal": "SIGKILL",
+    "forced": false
+  },
+  "before": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "RUNNING",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:21.256Z",
+    "finished_at": null,
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "6a1d383e-e5f5-47f0-b648-984b40d32710",
+    "lease_expires_at": "2026-08-28T05:00:26.256Z"
+  },
+  "recovered": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:21.256Z",
+    "finished_at": "2026-08-28T05:00:26.256Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "6a1d383e-e5f5-47f0-b648-984b40d32710",
+    "lease_expires_at": "2026-08-28T05:00:26.256Z"
+  },
+  "beforeExpiry": {
+    "pid": 84209,
+    "clock": "2026-08-28T05:00:26.255Z",
+    "recovered": []
+  },
+  "atExpiry": {
+    "pid": 84210,
+    "clock": "2026-08-28T05:00:26.256Z",
+    "recovered": [
+      {
+        "id": "11000000-0000-4000-a000-000000000001",
+        "monitorId": "11000000-0000-4000-9000-000000000001",
+        "trigger": "MANUAL",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": null,
+        "startedAt": "2026-08-28T05:00:21.256Z",
+        "finishedAt": "2026-08-28T05:00:26.256Z"
+      }
+    ]
+  },
+  "staleCompletionWrites": 0,
+  "originalIdentityReplay": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitorId": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "httpStatus": null,
+    "latencyMs": null,
+    "failureReason": null,
+    "startedAt": "2026-08-28T05:00:21.256Z",
+    "finishedAt": "2026-08-28T05:00:26.256Z"
+  },
+  "retryId": "6955e5a2-fe9f-4479-a45c-54f9260207e2",
+  "outboundCalls": 0,
+  "recoveryProcess": "separate Node process invoking the production recoverExpiredChecks operation; the real CLI worker loop invokes the same operation",
+  "durationMs": 1029
+}
diff --git a/evidence/phase-1/E11/author-recovery-1-held_response.json b/evidence/phase-1/E11/author-recovery-1-held_response.json
new file mode 100644
index 0000000..84f6578
--- /dev/null
+++ b/evidence/phase-1/E11/author-recovery-1-held_response.json
@@ -0,0 +1,90 @@
+{
+  "result": "PASS",
+  "checkpoint": {
+    "name": "held_response",
+    "schema": "e11_held_response",
+    "barrier": "fixture received GET /ok and withholds all HTTP headers",
+    "outboundCalls": 1,
+    "recoveredState": "ABORTED"
+  },
+  "observerPid": 84207,
+  "workerPid": 84213,
+  "workerId": "ba5c3a11-a591-4b4c-93c2-6bdcb16a717f",
+  "exit": {
+    "code": null,
+    "signal": "SIGKILL",
+    "forced": false
+  },
+  "before": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "RUNNING",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:22.204Z",
+    "finished_at": null,
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "ba5c3a11-a591-4b4c-93c2-6bdcb16a717f",
+    "lease_expires_at": "2026-08-28T05:00:27.204Z"
+  },
+  "recovered": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:22.204Z",
+    "finished_at": "2026-08-28T05:00:27.204Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "ba5c3a11-a591-4b4c-93c2-6bdcb16a717f",
+    "lease_expires_at": "2026-08-28T05:00:27.204Z"
+  },
+  "beforeExpiry": {
+    "pid": 84214,
+    "clock": "2026-08-28T05:00:27.203Z",
+    "recovered": []
+  },
+  "atExpiry": {
+    "pid": 84215,
+    "clock": "2026-08-28T05:00:27.204Z",
+    "recovered": [
+      {
+        "id": "11000000-0000-4000-a000-000000000001",
+        "monitorId": "11000000-0000-4000-9000-000000000001",
+        "trigger": "MANUAL",
+        "state": "ABORTED",
+        "httpStatus": null,
+        "latencyMs": null,
+        "failureReason": null,
+        "startedAt": "2026-08-28T05:00:22.204Z",
+        "finishedAt": "2026-08-28T05:00:27.204Z"
+      }
+    ]
+  },
+  "staleCompletionWrites": 0,
+  "originalIdentityReplay": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitorId": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "ABORTED",
+    "httpStatus": null,
+    "latencyMs": null,
+    "failureReason": null,
+    "startedAt": "2026-08-28T05:00:22.204Z",
+    "finishedAt": "2026-08-28T05:00:27.204Z"
+  },
+  "retryId": null,
+  "outboundCalls": 1,
+  "recoveryProcess": "separate Node process invoking the production recoverExpiredChecks operation; the real CLI worker loop invokes the same operation",
+  "durationMs": 901
+}
diff --git a/evidence/phase-1/E11/author-recovery-1-shutdown.json b/evidence/phase-1/E11/author-recovery-1-shutdown.json
new file mode 100644
index 0000000..ed044dd
--- /dev/null
+++ b/evidence/phase-1/E11/author-recovery-1-shutdown.json
@@ -0,0 +1,49 @@
+{
+  "result": "PASS",
+  "workerPid": 84221,
+  "workerId": "b3aede45-74c0-4423-9139-20e9db835274",
+  "signal": "SIGTERM",
+  "policy": "stop admission at signal; drain only current HTTP check; exit nonzero at 3000ms if drain or database close is still stuck, leaving uncertainty for lease recovery",
+  "signalAcknowledgedBeforeRelease": true,
+  "exit": {
+    "code": 0,
+    "signal": null,
+    "forced": false
+  },
+  "durationMs": 7,
+  "completed": {
+    "id": "11000000-0000-4000-a000-000000000001",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "SUCCEEDED",
+    "http_status": 200,
+    "latency_ms": 5,
+    "failure_reason": null,
+    "started_at": "2026-08-28T05:00:24.585Z",
+    "finished_at": "2026-08-28T05:00:24.594Z",
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-1",
+    "worker_id": "b3aede45-74c0-4423-9139-20e9db835274",
+    "lease_expires_at": "2026-08-28T05:00:29.585Z"
+  },
+  "queuedSecondUnchanged": {
+    "id": "11000000-0000-4000-a000-000000000002",
+    "monitor_id": "11000000-0000-4000-9000-000000000001",
+    "trigger": "MANUAL",
+    "state": "QUEUED",
+    "http_status": null,
+    "latency_ms": null,
+    "failure_reason": null,
+    "started_at": null,
+    "finished_at": null,
+    "queued_at": "2026-08-28T00:00:00.000Z",
+    "scheduled_for": null,
+    "request_user_id": "11000000-0000-4000-8000-000000000001",
+    "idempotency_key": "manual-recovery-e11-second",
+    "worker_id": null,
+    "lease_expires_at": null
+  },
+  "outboundCalls": 1
+}
diff --git a/evidence/phase-1/E11/terminal-write-failure.txt b/evidence/phase-1/E11/terminal-write-failure.txt
new file mode 100644
index 0000000..db7dc5c
--- /dev/null
+++ b/evidence/phase-1/E11/terminal-write-failure.txt
@@ -0,0 +1,30 @@
+Source: exact safe excerpt of the original author's execution, supplied by root
+from the stopped-author tool-output handoff. This is not a root or repair2 rerun.
+Command as recorded in root E11-attempt2.json:
+fnm exec --using24.19.0 npm run test:recovery
+Exit code: 1; 5 tests, 4 passed, 1 failed.
+Suite duration: 4060.529084ms; before_commit case duration: 551.9235ms.
+
+AssertionError [ERR_ASSERTION]: Expected values to be strictly deep-equal:
++ actual - expected
+ {
+  failure_reason: null,
++ finished_at: 2026-08-28T05:00:23.253Z,
++ http_status: 200,
+- finished_at: null,
+- http_status: null,
+  id: '11000000-0000-4000-a000-000000000001',
+  idempotency_key: 'manual-recovery-e11-1',
++ latency_ms: 4,
+- latency_ms: null,
+  lease_expires_at: 2026-08-28T05:00:28.245Z,
+  monitor_id: '11000000-0000-4000-9000-000000000001',
+  queued_at: 2026-08-28T00:00:00.000Z,
+  request_user_id: '11000000-0000-4000-8000-000000000001',
+  scheduled_for: null,
+  started_at: 2026-08-28T05:00:23.245Z,
++ state: 'SUCCEEDED',
+- state: 'RUNNING',
+  trigger: 'MANUAL',
+  worker_id: '26db8104-b9d9-4f2a-ac41-13bbe0c5067e'
+ }
diff --git a/evidence/phase-1/E11/verification-attempt3.json b/evidence/phase-1/E11/verification-attempt3.json
new file mode 100644
index 0000000..6fe87fa
--- /dev/null
+++ b/evidence/phase-1/E11/verification-attempt3.json
@@ -0,0 +1,279 @@
+{
+  "thread": "E11",
+  "profile": "phase-1",
+  "branch": "track/fundamentals-fastify",
+  "specRevision": "2ada57a71cd34fa2fae9809415c362a8bbfcdf02",
+  "attempt": 3,
+  "freshRepair": 2,
+  "threadStart": "591095c24b25efe59258ff07ee93e1b40adeadd4",
+  "repairStart": "edfb2821284ebf36096ce2203054d488cf7be369",
+  "result": "CANDIDATE_AWAITING_ROOT_VERIFICATION",
+  "provenance": {
+    "originalAuthor": "fundamentals_e08_a1",
+    "repairAgent": "fastify_e11_terminal_repair2",
+    "adoptedWork": "All 13 product, migration, test and configuration files already existed at dispatch and still exactly match the root-preserved attempt2 snapshot. This repair independently reviewed and committed that work; it did not author the existing terminal-transaction correction or alter product/test source.",
+    "rootFailureRecord": {
+      "path": "index/profiles/phase-1/verification/fundamentals-fastify/E11-attempt2.json",
+      "sha256": "cb55bef581d256d8da0c1f51db6bb67edb33f740fe81d688eb5e39852c94e715"
+    },
+    "rootPreservationManifest": {
+      "path": "index/profiles/phase-1/preserved/fundamentals-E11-attempt2/manifest.json",
+      "sha256": "870c717efe88d54f85e4fce45ed1e1957c67dcfb5aaa8215e0803cb1028119e7"
+    },
+    "rootPreservedPreCorrectionDiff": "index/profiles/phase-1/verification/fundamentals-fastify/E11-artifacts/root-observed-pre-terminal-fix.diff",
+    "adoptedSourceHashes": {
+      ".github/workflows/check.yml": "cdb9ad6894efd60a53c27d77ec1107c48a032ddb5889eb04615a479a1df68baf",
+      "app/monitors/monitor-workspace.tsx": "744a0085a9281f4c995355d26562de0bc7f5cf0b3ae3e2277f3e273ccb944001",
+      "package.json": "aa3403dc2c2206ac54d5b4fa74d7c1f843c1b0e8c13138a0383b565f4c318619",
+      "server/check.ts": "6532cea3ba3db1af746890d0bdc10903d2bb8d3cd4425feb990626f5605f24a2",
+      "server/migrate.ts": "0210d2809ed6aefab4147ae6aaeb79b2ea74e6b0971c14a706f2aba6f0aa315e",
+      "server/migrations/008_check_lease.sql": "4c9346ec915c848688864c6209093dadad8459e0860996310a4bb21b3267dfd2",
+      "server/model.ts": "271366ec0b3043b8e58b59edc691e721d0dad82eb66c73b46add54e5aea16f34",
+      "server/schema.ts": "f1fd635c6879c06214b4a911771b97a092744812bcc118782f62431e773dfa47",
+      "server/worker.ts": "394f9091fdff428b5c66777cb86aab6db975caefbe66f85b43d238ce50e83d3a",
+      "test/contracts.test.ts": "1a9bb99877c9f73bbcc47ad026c4d3e6391b28a77e262ddc7208c28e43d1f747",
+      "test/persistence.test.ts": "1fe23518050742d122db59d662f240c97d3ff03c739d8258d3781a4136393876",
+      "test/recovery.test.ts": "1167721f65b8750e821e9e9149ea28c08b5f140d367ef35e6915921468640920",
+      "test/worker.ts": "5aa8cc96647ba5b6ef2afc70749c0e448eb0cb77ee512e7a1bbbfa79eea1d327"
+    }
+  },
+  "immutableEvidence": {
+    "basis": "Every original E11 evidence file equals repair-start Git bytes; no scenario, fixture, baseline, observer runner or earlier failure record was edited.",
+    "sha256": {
+      "baseline-attempt1.json": "30f019be3e0fd568a8346711a98dfa4743e797c583aded89acdd0ac0a318a0ec",
+      "baseline-observer.json": "825e388c88545a1d40691c952130244f5bd2c79080190c292e5c5f7c1361f7a4",
+      "baseline-observer.mjs": "0f370c1c712618ad36faac8c8e023fb6b4d344af868809cfe0face385d3510d2",
+      "baseline.mjs": "2aa23e11221779c4130ea46a32ae04825b9a01c2d3c48bfe503ebe35da4a3047",
+      "fixture.ts": "52ad25d97c3703205fde226fa79541f80bbeaca3d987bb9da290b9340e22f539",
+      "observer-repair-freeze.json": "b8ee4a15fda360a78df46fd86a8f1306b2dd8cdba9009ea55e0714d2fdbdafe8",
+      "observer-repair-result.json": "803aefe595d76f6d8504085089b0451c4053624058feb5dd0efd17cd31fc1c23",
+      "scenario.json": "8ac1422579b3756d9b0b6151b20a4b2f37829cc0dfab9ab5942d5a8b2a292ce5"
+    }
+  },
+  "failureHistory": {
+    "baselineInvocation1": {
+      "result": "FAILED",
+      "exitCode": 1,
+      "artifact": "baseline-attempt1.json",
+      "durationMs": 727,
+      "reason": "The initial observer called stop after SIGKILL, issuing SIGTERM. Original failed observer and actual artifact remain unchanged."
+    },
+    "baselineInvocation2": {
+      "result": "REPRODUCED",
+      "exitCode": 0,
+      "artifact": "baseline-observer.json",
+      "durationMs": 518,
+      "reason": "The corrected child-close observer recorded actual SIGKILL, zero outbound calls and a durable RUNNING row without a finite lease; product matched Thread START."
+    },
+    "firstAuthorTypecheck": {
+      "exitCode": 2,
+      "exactExcerpt": "test/contracts.test.ts(176,53): error TS18047: result.latencyMs is possibly null.",
+      "correction": "The original author narrowly required non-null latency in the legacy observed-terminal assertion."
+    },
+    "firstRecoverySuite": {
+      "exitCode": 1,
+      "tests": 5,
+      "passed": 4,
+      "failed": 1,
+      "suiteDurationMs": 4060.529084,
+      "failedCase": "before_commit",
+      "failedCaseDurationMs": 551.9235,
+      "exactSafeFailureExcerpt": "terminal-write-failure.txt",
+      "attribution": "Author execution and native durations from root's stopped-author handoff, not a root or fresh-repair rerun.",
+      "failure": {
+        "checkpoint": "before_commit",
+        "reason": "After actual SIGKILL while terminal UPDATE was blocked, releasing the database row lock allowed pending autocommit UPDATE to persist SUCCEEDED200. The fixed checkpoint required the old RUNNING row to remain until safe ABORTED expiry recovery.",
+        "expected_after_kill": {
+          "state": "RUNNING",
+          "http_status": null,
+          "latency_ms": null,
+          "finished_at": null
+        },
+        "reported_actual_after_kill": {
+          "state": "SUCCEEDED",
+          "http_status": 200,
+          "latency_ms": 4,
+          "finished_at": "2026-08-28T05:00:23.253Z"
+        },
+        "worker_id": "26db8104-b9d9-4f2a-ac41-13bbe0c5067e",
+        "lease_expires_at": "2026-08-28T05:00:28.245Z"
+      },
+      "correction": "The original author replaced the pending autocommit completion with explicit BEGIN/UPDATE/post-lock lease recheck/COMMIT before this repair was dispatched."
+    },
+    "targetedAuthorRecheck": {
+      "exitCode": 0,
+      "tests": 1,
+      "passed": 1,
+      "failed": 0,
+      "suiteDurationMs": 1211.021333,
+      "caseDurationMs": 1009.62375,
+      "artifact": "author-before_commit-recheck.json",
+      "attribution": "Completed by the original author before interruption; actual output bytes inspected by root and this repair."
+    },
+    "secondAuthorTypecheck": {
+      "exitCode": 0,
+      "durationMs": null,
+      "attribution": "Started by the original author before interruption; root directly read the successful final output. Elapsed time was not instrumented."
+    }
+  },
+  "authorCommandsAsRecordedByRoot": [
+    {
+      "command": "fnm exec --using24.19.0 npm run typecheck",
+      "exit_code": 2,
+      "duration_seconds": null,
+      "failure": "test/contracts.test.ts(176,53): error TS18047: result.latencyMs is possibly null."
+    },
+    {
+      "command": "fnm exec --using24.19.0 npm run test:recovery",
+      "exit_code": 1,
+      "tests": 5,
+      "passed": 4,
+      "failed": 1,
+      "suite_duration_ms": 4060.529084,
+      "failing_case": "before_commit",
+      "failing_case_duration_ms": 551.9235
+    },
+    {
+      "command": "fnm exec --using24.19.0 node --test --test-name-pattern='E11 SIGKILL before_commit' test/recovery.test.ts",
+      "exit_code": 0,
+      "tests": 1,
+      "passed": 1,
+      "suite_duration_ms": 1211.021333,
+      "case_duration_ms": 1009.62375,
+      "occurred_before_root_interruption": true
+    },
+    {
+      "command": "fnm exec --using24.19.0 npm run typecheck",
+      "exit_code": 0,
+      "duration_seconds": null,
+      "execution_started_before_root_interruption": true,
+      "final_read_only_poll_session": 36185
+    }
+  ],
+  "actualAuthorArtifacts": [
+    {
+      "file": "author-recovery-1-before_io.json",
+      "source": "output/phase-1/e11/before_io.json",
+      "sha256": "aa038621f7736357efcfe4fd07d46a921330b259d0b3c1b8168c6ee4a861ebe0",
+      "origin": "Original author first full recovery suite",
+      "result": "PASS",
+      "observedDurationMs": 1029
+    },
+    {
+      "file": "author-recovery-1-held_response.json",
+      "source": "output/phase-1/e11/held_response.json",
+      "sha256": "dd04173f169ec471392f1050fb22247255a6d6a3a568c58a9af660ba83e935b0",
+      "origin": "Original author first full recovery suite",
+      "result": "PASS",
+      "observedDurationMs": 901
+    },
+    {
+      "file": "author-recovery-1-after_commit.json",
+      "source": "output/phase-1/e11/after_commit.json",
+      "sha256": "b54847f2849385dcebb397512e230255771b69947f7059e50f903e09e37d51c9",
+      "origin": "Original author first full recovery suite",
+      "result": "PASS",
+      "observedDurationMs": 929
+    },
+    {
+      "file": "author-recovery-1-shutdown.json",
+      "source": "output/phase-1/e11/shutdown.json",
+      "sha256": "220741028e7227a7eacb724859d9c7246ee5ed9579cf75827f1ac09bab2572b8",
+      "origin": "Original author first full recovery suite",
+      "result": "PASS",
+      "observedDurationMs": 7
+    },
+    {
+      "file": "author-before_commit-recheck.json",
+      "source": "output/phase-1/e11/before_commit.json",
+      "sha256": "a4dfc159e8c84279c457801d67037b558174e04311f1908f3c4c31bbe6b93f32",
+      "origin": "Original author targeted correction recheck; not the initial full suite",
+      "result": "PASS",
+      "observedDurationMs": 1001
+    }
+  ],
+  "repairVerification": {
+    "productOrTestEdits": 0,
+    "baselineInvocations": 0,
+    "recoverySuiteInvocations": 0,
+    "targetedRecoveryInvocations": 0,
+    "typecheckInvocations": 0,
+    "unitInvocations": 0,
+    "functionalInvocations": 0,
+    "integrationInvocations": 0,
+    "ownershipInvocations": 0,
+    "productionBuildInvocations": 0,
+    "browserInvocations": 0,
+    "executionInvocations": 0,
+    "readOnlyAudit": {
+      "result": "PASS",
+      "node": "24.19.0",
+      "toolChunkId": "872cab",
+      "toolWallSeconds": 0.338982834,
+      "checks": [
+        "Branch, HEAD and SPEC_REVISION exact",
+        "13 preserved source/test files match manifest",
+        "7 existing output files match manifest",
+        "8 original E11 evidence files equal repair-start Git bytes"
+      ]
+    },
+    "sourceReview": [
+      "Claim persists a five-second lease using the same database timestamp as started_at before outbound I/O; old RUNNING rows receive a finite lease in migration008.",
+      "Completion obtains one PoolClient, begins a transaction, matches ID/RUNNING/owner/live lease, rechecks the lease after the UPDATE lock wait, then commits; any absent/expired match rolls back.",
+      "Recovery updates only expired RUNNING rows to ABORTED, clears unobserved endpoint fields and preserves identity/ownership evidence. It never requeues or performs outbound I/O.",
+      "The real worker loop calls production recoverExpiredChecks using the database clock. The fixed exact-expiry observer is a separate Node test consumer of that same operation, not a continuously running CLI worker.",
+      "The frozen SIGTERM consumer waits for stopping acknowledgement before releasing its held200; only the first run can drain, with the second unchanged QUEUED.",
+      "The three-second stuck-shutdown timer exits nonzero and leaves uncommitted uncertainty; this deadline branch was source-inspected only, not separately executed.",
+      "Terminal ABORTED values remain null on the wire and display Unknown in the latest result/history; existing filters and polling termination remain intact."
+    ]
+  },
+  "budget": {
+    "totalBaselineInvocations": 2,
+    "failedObserverBaselines": 1,
+    "correctedReproducedBaselines": 1,
+    "totalAuthorTypecheckInvocations": 2,
+    "typecheckFailures": 1,
+    "recoveryFullSuites": 1,
+    "beforeCommitExtraRechecks": 1,
+    "perCaseInvocationsBeforeRootFinal": {
+      "before_io": 1,
+      "held_response": 1,
+      "before_commit": 2,
+      "after_commit": 1,
+      "shutdown": 1
+    },
+    "rootFinalRecoverySuiteReserved": 1,
+    "perCaseTotalsAfterReservedRootSuite": {
+      "before_io": 2,
+      "held_response": 2,
+      "before_commit": 3,
+      "after_commit": 2,
+      "shutdown": 2
+    },
+    "freshRepairsUsed": 2,
+    "maximumFreshRepairs": 2,
+    "furtherRepairAllowance": 0,
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0,
+    "note": "The five passing copied artifacts do not represent one passing corrected full suite: four are from the failed first suite and before_commit is from its single targeted correction recheck. Root owns the remaining final corrected full-suite verification."
+  },
+  "scope": {
+    "newDependencies": [],
+    "earlierRuntimePinsLockfileAndMigrationsUnchanged": true,
+    "mainSpecIndexThreadDocsTagsUntouched": true,
+    "otherTrackSourceAccessed": false,
+    "newProductOrTestEditsByThisRepair": false,
+    "baselineOrRecoveryRerunByThisRepair": false,
+    "noCredentialsSessionTokensPrivateBodiesOrBrowserArtifacts": true,
+    "pushPerformed": false
+  },
+  "cleanup": {
+    "repairApplicationWorkerOrFixtureProcessesStarted": 0,
+    "repairSchemasCreated": 0,
+    "basis": "No application, fixture, worker, database schema or browser was started by this repair. The preserved recovery consumers close only their owned processes/schemas. Root retains the independent final cleanup gate."
+  },
+  "unresolved": [
+    "Root final unit/API/PostgreSQL/ownership/recovery/production-browser/execution verification is still pending; this candidate is not a verified E11 completion. The original author's final typecheck passed and its source remains byte-identical."
+  ]
+}
diff --git a/package.json b/package.json
index ae88a3f..c6a4417 100644
--- a/package.json
+++ b/package.json
@@ -23,7 +23,8 @@
     "db:migrate": "node server/migrate.ts",
     "test:integration": "node --test --test-concurrency=1 test/persistence.test.ts test/storage-*.test.ts",
     "test:execution": "node --test test/execution.test.ts",
-    "test:ownership": "node --test test/ownership.test.ts"
+    "test:ownership": "node --test test/ownership.test.ts",
+    "test:recovery": "node --test test/recovery.test.ts"
   },
   "dependencies": {
     "fastify": "5.12.1",
diff --git a/test/recovery.test.ts b/test/recovery.test.ts
new file mode 100644
index 0000000..2c65fe5
--- /dev/null
+++ b/test/recovery.test.ts
@@ -0,0 +1,223 @@
+import { test } from 'node:test';
+import assert from 'node:assert/strict';
+import { execFile } from 'node:child_process';
+import { promisify } from 'node:util';
+import { mkdir, writeFile } from 'node:fs/promises';
+import { setTimeout as delay } from 'node:timers/promises';
+import type { Pool, PoolClient } from 'pg';
+import { buildApp } from '../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../server/database.ts';
+import type { DatabaseConfig } from '../server/database.ts';
+import { migrate } from '../server/migrate.ts';
+import { checkRunFromRow } from '../server/mapping.ts';
+import type { ObservedCheckRun } from '../server/model.ts';
+import { CHECK_LEASE_MS, SHUTDOWN_GRACE_MS, completeCheck } from '../server/worker.ts';
+import { authenticatedInject, loginForTest } from './auth.ts';
+import { startTestWorker, waitForTerminalCheck } from './worker.ts';
+import { holdBeforeIo, insertRecoveryFixture, recoveryFixture, requireFreePorts } from '../evidence/phase-1/E11/fixture.ts';
+import scenario from '../evidence/phase-1/E11/scenario.json' with { type: 'json' };
+
+const execute = promisify(execFile);
+async function bounded<T>(promise: Promise<T>, message: string) {
+  let timer: ReturnType<typeof setTimeout> | undefined;
+  try {
+    return await Promise.race([promise, new Promise<never>((_, reject) => {
+      timer = setTimeout(() => reject(new Error(message)), scenario.watchdogMs);
+    })]);
+  } finally { clearTimeout(timer); }
+}
+async function record(name: string, value: unknown) {
+  await mkdir('output/phase-1/e11', { recursive: true });
+  await writeFile(`output/phase-1/e11/${name}.json`, JSON.stringify(value, null, 2) + '\n');
+}
+async function row(pool: Pool, id = scenario.check.id) {
+  const result = await pool.query('SELECT * FROM check_runs WHERE id = $1', [id]);
+  assert.equal(result.rowCount, 1);
+  return result.rows[0];
+}
+async function recoverInAnotherProcess(config: DatabaseConfig, now: Date) {
+  const result = await execute(process.execPath, ['--input-type=module', '-e', `
+    import { databaseConfig, databasePool } from './server/database.ts';
+    import { recoverExpiredChecks } from './server/worker.ts';
+    const pool = databasePool(databaseConfig());
+    try {
+      const recovered = await recoverExpiredChecks(pool, new Date(process.argv[1]));
+      console.log(JSON.stringify({ pid: process.pid, clock: process.argv[1], recovered }));
+    } finally { await pool.end(); }
+  `, now.toISOString()], { env: { ...process.env, DATABASE_URL: config.connectionString, DATABASE_SCHEMA: config.schema },
+    timeout: scenario.watchdogMs });
+  return JSON.parse(result.stdout) as { pid: number; clock: string; recovered: { id: string; state: string }[] };
+}
+
+for (const checkpoint of scenario.checkpoints) {
+  test(`E11 SIGKILL ${checkpoint.name}: expiry cannot invent or reopen an endpoint outcome`, { timeout: 30000 }, async () => {
+    assert.equal(CHECK_LEASE_MS, scenario.leaseMs);
+    const began = performance.now();
+    const config = { ...databaseConfig(), schema: checkpoint.schema };
+    const pool = databasePool(config);
+    const fixture = recoveryFixture(true);
+    const app = buildApp(undefined, config);
+    let owned = false;
+    let barrier: Awaited<ReturnType<typeof holdBeforeIo>> | undefined;
+    let terminalLock: PoolClient | undefined;
+    let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
+    let exited = false;
+    try {
+      await requireFreePorts();
+      await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true;
+      await migrate(config);
+      const credentials = await insertRecoveryFixture(pool);
+      await new Promise<void>((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+      if (checkpoint.name === 'before_io') barrier = await holdBeforeIo(pool);
+      const applicationName = `e11_${checkpoint.name}_worker`;
+      const workerUrl = new URL(config.connectionString);
+      workerUrl.searchParams.set('application_name', applicationName);
+      worker = await startTestWorker({ ...config, connectionString: workerUrl.href });
+      assert.notEqual(worker.pid, process.pid);
+      if (barrier) await barrier.arrived(applicationName);
+      else {
+        await bounded(fixture.entered, 'Held HTTP request did not arrive.');
+        if (checkpoint.name === 'before_commit') {
+          terminalLock = await pool.connect();
+          await terminalLock.query('BEGIN');
+          await terminalLock.query('SELECT id FROM check_runs WHERE id = $1 FOR UPDATE', [scenario.check.id]);
+          const lockPid = (await terminalLock.query<{ pid: number }>('SELECT pg_backend_pid() AS pid')).rows[0].pid;
+          fixture.release();
+          const deadline = Date.now() + scenario.watchdogMs;
+          let blocked = false;
+          while (Date.now() < deadline) {
+            const waiting = await pool.query(`SELECT pid FROM pg_stat_activity WHERE application_name = $1
+              AND wait_event_type = 'Lock' AND $2 = ANY(pg_blocking_pids(pid))
+              AND query LIKE 'UPDATE check_runs SET state = $2,%'`, [applicationName, lockPid]);
+            if (waiting.rowCount === 1) { blocked = true; break; }
+            await delay(scenario.observationPollMs);
+          }
+          assert.equal(blocked, true, 'HTTP200 must be observed before the blocked terminal write.');
+        } else if (checkpoint.name === 'after_commit') {
+          fixture.release();
+          assert.equal((await waitForTerminalCheck(pool, scenario.check.id)).state, 'SUCCEEDED');
+        }
+      }
+      const before = await row(pool);
+      assert.equal(before.worker_id, worker.workerId);
+      assert.equal(before.lease_expires_at.getTime() - before.started_at.getTime(), scenario.leaseMs);
+      assert.equal(before.state, checkpoint.name === 'after_commit' ? 'SUCCEEDED' : 'RUNNING');
+      assert.equal(fixture.calls.get('/ok') ?? 0, checkpoint.outboundCalls);
+      assert.equal(process.kill(worker.pid, 'SIGKILL'), true);
+      const exit = await bounded(worker.waitForExit(), 'Killed worker did not close.'); exited = true;
+      assert.deepEqual(exit, { code: null, signal: 'SIGKILL', forced: false });
+      if (barrier) { await barrier.release(); barrier = undefined; }
+      if (terminalLock) { await terminalLock.query('ROLLBACK'); terminalLock.release(); terminalLock = undefined; }
+      assert.deepEqual(await row(pool), before);
+
+      const beforeExpiry = await recoverInAnotherProcess(config, new Date(before.lease_expires_at.getTime() - 1));
+      assert.notEqual(beforeExpiry.pid, process.pid);
+      assert.notEqual(beforeExpiry.pid, worker.pid);
+      assert.deepEqual(beforeExpiry.recovered, []);
+      assert.deepEqual(await row(pool), before);
+      const atExpiry = await recoverInAnotherProcess(config, before.lease_expires_at);
+      assert.notEqual(atExpiry.pid, process.pid);
+      assert.notEqual(atExpiry.pid, worker.pid);
+      assert.equal(atExpiry.recovered.length, checkpoint.recoveredState === 'ABORTED' ? 1 : 0);
+      const recovered = await row(pool);
+      assert.equal(recovered.id, scenario.check.id);
+      assert.equal(recovered.state, checkpoint.recoveredState);
+      if (recovered.state === 'ABORTED') {
+        assert.deepEqual(atExpiry.recovered.map(check => check.id), [scenario.check.id]);
+        assert.deepEqual({ httpStatus: recovered.http_status, latencyMs: recovered.latency_ms, failureReason: recovered.failure_reason }, scenario.recovery.unknownFields);
+        assert.equal(recovered.finished_at.toISOString(), atExpiry.clock);
+      } else assert.deepEqual(recovered, before);
+      const stale: ObservedCheckRun = { id: scenario.check.id, monitorId: scenario.monitor.id, trigger: 'MANUAL',
+        state: 'SUCCEEDED', httpStatus: 200, latencyMs: 0, failureReason: null,
+        startedAt: before.started_at.toISOString(), finishedAt: before.lease_expires_at.toISOString() };
+      assert.equal(await completeCheck(pool, worker.workerId, stale), null);
+      assert.deepEqual(await row(pool), recovered);
+      assert.equal(fixture.calls.get('/ok') ?? 0, checkpoint.outboundCalls);
+
+      const inject = authenticatedInject(app, await loginForTest(app, credentials));
+      const path = `/monitors/${scenario.monitor.id}/checks`;
+      const replay = await inject({ method: 'POST', url: path, headers: { 'idempotency-key': scenario.check.key } });
+      assert.equal(replay.statusCode, 202);
+      assert.deepEqual(replay.json().data, checkRunFromRow(recovered));
+      const history = await inject(path);
+      assert.equal(history.statusCode, 200);
+      assert.deepEqual(history.json().data.items, [checkRunFromRow(recovered)]);
+      let retryId: string | null = null;
+      if (checkpoint.name === 'before_io') {
+        const retry = await inject({ method: 'POST', url: path, headers: { 'idempotency-key': scenario.check.newIntentKey } });
+        assert.equal(retry.statusCode, 202);
+        assert.equal(retry.json().data.state, 'QUEUED');
+        retryId = retry.json().data.id;
+        assert.notEqual(retryId, scenario.check.id);
+        assert.deepEqual(await row(pool), recovered);
+        assert.equal((await pool.query('SELECT count(*)::int AS count FROM check_runs')).rows[0].count, 2);
+      }
+      await record(checkpoint.name, { result: 'PASS', checkpoint, observerPid: process.pid, workerPid: worker.pid,
+        workerId: worker.workerId, exit, before, recovered, beforeExpiry, atExpiry, staleCompletionWrites: 0,
+        originalIdentityReplay: replay.json().data, retryId, outboundCalls: fixture.calls.get('/ok') ?? 0,
+        recoveryProcess: scenario.recovery.process, durationMs: Math.round(performance.now() - began) });
+    } finally {
+      if (worker && !exited) await worker.stop();
+      if (barrier) await barrier.release();
+      if (terminalLock) { await terminalLock.query('ROLLBACK'); terminalLock.release(); }
+      await app.close();
+      fixture.release();
+      if (fixture.server.listening) { fixture.server.closeAllConnections(); await new Promise<void>(resolve => fixture.server.close(() => resolve())); }
+      if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+      await pool.end();
+      await requireFreePorts();
+    }
+  });
+}
+
+test('E11 SIGTERM drains the current HTTP check and never claims the queued second run', { timeout: 30000 }, async () => {
+  assert.equal(SHUTDOWN_GRACE_MS, scenario.shutdown.graceMs);
+  const config = { ...databaseConfig(), schema: scenario.shutdown.schema };
+  const pool = databasePool(config);
+  const fixture = recoveryFixture(true);
+  let owned = false;
+  let worker: Awaited<ReturnType<typeof startTestWorker>> | undefined;
+  let exited = false;
+  try {
+    await requireFreePorts();
+    await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`); owned = true;
+    await migrate(config);
+    await insertRecoveryFixture(pool);
+    await new Promise<void>((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+    worker = await startTestWorker(config);
+    await bounded(fixture.entered, 'Shutdown in-flight request did not arrive.');
+    assert.equal((await row(pool)).state, 'RUNNING');
+    await pool.query(`INSERT INTO check_runs (id, monitor_id, trigger, state, queued_at, request_user_id, idempotency_key)
+      VALUES ($1, $2, 'MANUAL', 'QUEUED', $3, $4, $5)`, [scenario.shutdown.secondCheckId, scenario.monitor.id,
+      new Date(scenario.check.queuedAt), scenario.user.id, scenario.shutdown.secondKey]);
+    const queued = await row(pool, scenario.shutdown.secondCheckId);
+    const began = performance.now();
+    assert.equal(process.kill(worker.pid, 'SIGTERM'), true);
+    await bounded(worker.stopping, 'Worker did not acknowledge SIGTERM.');
+    fixture.release();
+    const exit = await bounded(worker.waitForExit(), 'Worker did not finish its bounded drain.'); exited = true;
+    const durationMs = Math.round(performance.now() - began);
+    assert.equal(exit.code, scenario.shutdown.expectedExitCode);
+    assert.equal(exit.signal, null);
+    assert.equal(exit.forced, false);
+    assert.ok(durationMs <= scenario.shutdown.graceMs);
+    const completed = await row(pool);
+    assert.equal(completed.state, scenario.shutdown.expectedFirstState);
+    assert.equal(completed.http_status, 200);
+    assert.equal(completed.worker_id, worker.workerId);
+    assert.deepEqual(await row(pool, scenario.shutdown.secondCheckId), queued);
+    assert.equal(queued.state, scenario.shutdown.expectedSecondState);
+    assert.equal(queued.worker_id, null);
+    assert.equal(fixture.calls.get('/ok'), scenario.shutdown.expectedOutboundCalls);
+    await record('shutdown', { result: 'PASS', workerPid: worker.pid, workerId: worker.workerId, signal: 'SIGTERM',
+      policy: scenario.shutdown.policy, signalAcknowledgedBeforeRelease: true, exit, durationMs, completed,
+      queuedSecondUnchanged: queued, outboundCalls: fixture.calls.get('/ok') });
+  } finally {
+    if (worker && !exited) await worker.stop();
+    fixture.release();
+    if (fixture.server.listening) { fixture.server.closeAllConnections(); await new Promise<void>(resolve => fixture.server.close(() => resolve())); }
+    if (owned) await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+    await pool.end();
+    await requireFreePorts();
+  }
+});
diff --git a/test/worker.ts b/test/worker.ts
index ce6c253..45f7aba 100644
--- a/test/worker.ts
+++ b/test/worker.ts
@@ -13,6 +13,7 @@ export async function startTestWorker(config: DatabaseConfig) {
     stdio: ['ignore', 'pipe', 'pipe'],
   });
   const exited = new Promise<{ code: number | null; signal: string | null }>((resolve) => child.once('close', (code, signal) => resolve({ code, signal })));
+  const stopping = Promise.withResolvers<void>();
   let workerId = '';
   async function stop() {
     let forced = false;
@@ -32,6 +33,7 @@ export async function startTestWorker(config: DatabaseConfig) {
       let output = '';
       child.stdout.on('data', (chunk) => {
         output += String(chunk);
+        if (output.includes('Check worker stopping.')) stopping.resolve();
         const ready = /Check worker ready\. ([0-9a-f-]{36})/.exec(output);
         if (ready) { workerId = ready[1]; clearTimeout(timer); resolve(); }
       });
@@ -39,7 +41,7 @@ export async function startTestWorker(config: DatabaseConfig) {
       child.once('exit', () => { clearTimeout(timer); reject(new Error('Worker exited before readiness.')); });
     });
   } catch (error) { await stop(); throw error; }
-  return { pid: child.pid!, workerId, stop, waitForExit };
+  return { pid: child.pid!, workerId, stop, waitForExit, stopping: stopping.promise };
 }
 
 export async function waitForTerminalCheck(pool: Pool, id: string): Promise<TerminalCheckRun> {
@@ -48,7 +50,7 @@ export async function waitForTerminalCheck(pool: Pool, id: string): Promise<Term
     const result = await pool.query<CheckRunRow>('SELECT * FROM check_runs WHERE id = $1', [id]);
     assert.ok(result.rows[0], 'The accepted execution must remain persisted.');
     const check = checkRunFromRow(result.rows[0]);
-    if (check.state === 'SUCCEEDED' || check.state === 'FAILED') return check as TerminalCheckRun;
+    if (check.state === 'SUCCEEDED' || check.state === 'FAILED' || check.state === 'ABORTED') return check as TerminalCheckRun;
     await delay(25);
   }
   throw new Error('Worker did not persist a terminal result before the test deadline.');
