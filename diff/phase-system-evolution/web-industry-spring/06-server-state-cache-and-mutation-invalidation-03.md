## `test(state): preserve E06 response-barrier verification`

diff --git a/evidence/E06/browser-outcomes.json b/evidence/E06/browser-outcomes.json
new file mode 100644
index 0000000..47b0e0e
--- /dev/null
+++ b/evidence/E06/browser-outcomes.json
@@ -0,0 +1,38 @@
+[
+  {
+    "source": "output/e06/baseline-report.json",
+    "stats": {
+      "startTime": "2026-08-28T02:20:15.129Z",
+      "duration": 12276.28,
+      "expected": 0,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Decisive unchanged-START failure: two form submits produced two real create POSTs and two durable rows; later500 probes were not run."
+  },
+  {
+    "source": "output/e06/targeted-browser-report.json",
+    "stats": {
+      "startTime": "2026-08-28T02:25:41.311Z",
+      "duration": 12695.581,
+      "expected": 0,
+      "skipped": 0,
+      "unexpected": 1,
+      "flaky": 0
+    },
+    "observation": "Authoring observation failure: the product alert and Next route announcer both matched an unscoped locator after injected update500. Later no-write/delete/release assertions were not completed."
+  },
+  {
+    "source": "output/e06/targeted-browser-2-report.json",
+    "stats": {
+      "startTime": "2026-08-28T02:27:10.940Z",
+      "duration": 14611.32,
+      "expected": 1,
+      "skipped": 0,
+      "unexpected": 0,
+      "flaky": 0
+    },
+    "observation": "Complete fixed scenario passed after scoping both alert locators to the existing product data-error-code attribute. Inputs, barriers and required assertions unchanged."
+  }
+]
diff --git a/evidence/E06/server-state.json b/evidence/E06/server-state.json
new file mode 100644
index 0000000..00cad4f
--- /dev/null
+++ b/evidence/E06/server-state.json
@@ -0,0 +1,28 @@
+{
+  "mode": "post-change regression",
+  "start": "ef470a301358932a77457d714c79494e631a8f96",
+  "fixtureSha256": "62c32bb19c2f4cf1f8c958d72014595d11220763d3c4e5f450019c5c64420556",
+  "completed": [
+    "B create/edit/pause/resume/delete and list/detail reload",
+    "A one real Check, latest/history toggle and reload",
+    "update500/delete500 retain authority and independent create pending",
+    "release real201, one request/row, pending clears and reload remains canonical"
+  ],
+  "result": "PASS",
+  "delayedCreate": {
+    "submitEvents": 2,
+    "createPosts": 1,
+    "durableRows": 1,
+    "realResponseStatus": 201
+  },
+  "injectedFailures": {
+    "methods": [
+      "PUT",
+      "DELETE"
+    ],
+    "status": 500,
+    "whileCreateHeld": true,
+    "authoritativeRowsUnchanged": true
+  },
+  "heldRoutesSettled": true
+}
diff --git a/evidence/E06/verification-runs.jsonl b/evidence/E06/verification-runs.jsonl
new file mode 100644
index 0000000..1443454
--- /dev/null
+++ b/evidence/E06/verification-runs.jsonl
@@ -0,0 +1,5 @@
+{"command":"E06_BASELINE=1 npm run test:e2e -- tests/browser/server-state.spec.ts","kind":"baseline","startedAt":"2026-08-28T02:20:14.581Z","elapsedSeconds":13.11,"exitCode":1,"cleanupExitCode":0,"log":"output/e06/baseline.log"}
+{"command":"npm run typecheck","startedAt":"2026-08-28T02:24:45.658Z","elapsedSeconds":1.955,"exitCode":0,"log":"output/e06/typecheck.log"}
+{"command":"npm run test:e2e -- tests/browser/server-state.spec.ts","kind":"targeted-regression","startedAt":"2026-08-28T02:25:40.536Z","elapsedSeconds":13.815,"exitCode":1,"cleanupExitCode":0,"log":"output/e06/targeted-browser.log"}
+{"command":"npm run test:e2e -- tests/browser/server-state.spec.ts","kind":"targeted-regression","startedAt":"2026-08-28T02:27:10.340Z","elapsedSeconds":15.521,"exitCode":0,"cleanupExitCode":0,"log":"output/e06/targeted-browser-2.log"}
+{"command":"npm run build","startedAt":"2026-08-28T02:28:40.400Z","elapsedSeconds":12.008,"exitCode":0,"log":"output/e06/build.log"}
diff --git a/evidence/E06/verification.md b/evidence/E06/verification.md
new file mode 100644
index 0000000..b4f678b
--- /dev/null
+++ b/evidence/E06/verification.md
@@ -0,0 +1,77 @@
+# E06 verification — attempt1
+
+START: `ef470a301358932a77457d714c79494e631a8f96`  
+Spec: `0a006589477f8ae47bad3faa5510c999cff85ee4`  
+Frozen fixture SHA-256: `62c32bb19c2f4cf1f8c958d72014595d11220763d3c4e5f450019c5c64420556`
+
+## New invocations
+
+Exact commands, start times, exit codes and cleanup results are retained in
+`verification-runs.jsonl`; original browser reporter statistics and failure
+observations are retained in `browser-outcomes.json`.
+
+| Invocation | Result | Seconds |
+| --- | --- | ---: |
+| One unchanged-START E06 baseline | Decisive failure:2 submits→2 POSTs/2 durable rows | 13.110 |
+| `npm run typecheck` | Passed | 1.955 |
+| First targeted E06 regression | Failed only at an ambiguous alert locator after update500 | 13.815 |
+| Corrected targeted E06 regression | Complete fixed scenario passed | 15.521 |
+| `npm run build` | Production build and its final TypeScript check passed | 12.008 |
+
+The baseline first completed B's create/edit/pause/resume/delete and reload
+sequence, then A's real Check/latest/history sequence. It held one real create201
+after commit and observed2 requests/2 rows from exactly2 submit events. Reproduction
+stopped at that decisive failure; later500 probes were not run.
+
+The first post-change test reached an injected update500, then an unscoped alert
+locator matched both the product error and Next's route announcer. It did not
+complete the later authority/delete/release checks. Both locators were narrowed
+to the existing product `[role="alert"][data-error-code]` element. No product code,
+request, input, barrier, expected status or unchanged-authority assertion was
+altered for that correction. No retries were enabled.
+
+## Passing fixed scenario
+
+`server-state.json` records the complete pass:
+
+- B's create/update/pause/resume/delete refreshes the existing list/article detail;
+  reload retains canonical values and deleted detail returns404.
+- A's one real Check appears consistently as latest and in history across
+  hide/reopen and reload.
+- The first real create response is held after commit. Exactly two submit events
+  produce one POST and one durable `Pending C` row.
+- Exactly one update500 and one delete500 occur while create remains held. A's
+  canonical detail/latest/history stay unchanged, errors are visible, and neither
+  failure clears create pending. Injected failures do not reach the backend.
+- Releasing real201 clears pending and renders exactly one canonical row. Reload
+  keeps that row and A's original authoritative state/history.
+- Held route promises settle in `finally`; each browser invocation explicitly
+  removes only the disposable `e04_browser` schema using the existing helper.
+  Final listener inspection found4321–4325 free; public data and the database
+  volume were not touched.
+
+## Reused verification and scope
+
+The parent independently verified `progress/industry-spring/E05` at START,
+including36 Java tests, the backend/session/ownership boundary, real restart and
+occupied-port guard. A diff against START confirmed no changes to backend,
+migrations, API/auth client, dependencies, database/CI scripts or runtime/test
+configuration. Those unchanged backend gates are reused, not reported as new runs.
+The parent will run the full affected browser regression on the final candidate;
+this authoring pass ran only the fixed E06 browser case, static types and build.
+
+Production changes are confined to the Monitor page and its specific state hook.
+The hook owns canonical remote data and request phases; the page retains only
+transient editing/history visibility. Successful writes update their related
+cached data; failures do not mutate it. Request guards are synchronous and scoped
+to creation or a Monitor. No new route, dependency, backend policy, generic cache
+framework, optimistic-write requirement, E07 or E08 behavior was introduced.
+
+Runtime remains Node24.19.0/npm11.17.0 and the existing Java21/Maven3.9.11 backend.
+No credentials, session/CSRF values or password hashes are recorded. Existing
+browser traces/screenshots/videos/storage-state capture remain disabled.
+
+Budget: baseline1, post-change targeted browser invocations2, standalone typecheck1,
+production build1. Load0, automatic retries0, parameter sweeps0, formal repairs0.
+The parent-authorized reuse workflow keeps this E06 dispatch separate from E05;
+the fixed Spec-Revision trailer is unchanged. Stop after reporting this candidate.
