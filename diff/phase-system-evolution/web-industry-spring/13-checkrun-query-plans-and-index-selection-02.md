## `docs(e20): record the existing-index decision and bounded evidence`

diff --git a/TRACK.md b/TRACK.md
index 3e39bf2..a6b68a3 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -4,7 +4,7 @@ Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not
 
 Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
-E12 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
+E20 is a local development product: Next.js/React renders login, logout, Monitor lifecycle and Check history from request-scoped server data. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and Check execution intents/results. Manual requests have durable owner-scoped identities, and competing workers claim one execution owner before bounded HTTP/HTTPS I/O. Monitor and CheckRun access is scoped to the signed-in user. There is no signup, Redis, broker or production application container.
 
 ## Pinned toolchain
 
@@ -461,3 +461,26 @@ stubs; actual HTTP I/O uses the controlled local4325 fixtures. The public IPv6 H
 and TLS parameter assertions do not claim a live TLS handshake. No public network
 or metadata endpoint was contacted, and no baseline, E11 crash scenario, load run,
 parameter sweep or automatic retry was repeated during this repair.
+
+## Check history query plan (E20)
+
+The fixed 99,000-row dataset already satisfies the history plan criteria using
+the V5 `(monitor_id, state, finished_at DESC, id DESC)` index. The actual
+owner-scoped FAILED query fetches 21 rows for a public page of 20 plus its cursor.
+The single author baseline scanned 21 CheckRuns, removed none by filter, and
+needed no explicit Sort or CheckRun sequential scan. The first two pages retain
+all 40 expected records in order; Bob's read of Alice's Monitor remains 404.
+
+E20 adds no index, migration or production behavior. Existing indexes still
+cost storage and maintenance when executions are inserted or indexed fields
+change; a redundant index would add those costs without reducing this query's
+already bounded row fetch. The observed plan and index sizes are evidence for
+this fixed dataset, not a latency benchmark or a general workload claim.
+
+The capped dataset test is opt-in through `-De20.plan-proof=baseline` or `final`;
+default tests do not repeat it. It reuses the existing history SQL inspector and
+database helpers. Nonsecret bind TRACE is enabled only around the actual history
+read and restored afterward. Evidence, the sole author invocation and the frozen
+SQL/data are under `evidence/phase-1/E20`. At this author submission, root's one
+independent final EXPLAIN and related history regression are pending. No E11
+crash, E12 outbound, browser or load gate was repeated for E20.
diff --git a/evidence/phase-1/E20/README.md b/evidence/phase-1/E20/README.md
new file mode 100644
index 0000000..a39c580
--- /dev/null
+++ b/evidence/phase-1/E20/README.md
@@ -0,0 +1,111 @@
+# E20 author result — retain the existing index
+
+Profile `phase-1`, attempt 1. START
+`b309d2f8b6de8b81c5936906e296f314441646bc`; Spec-Revision
+`2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+The unchanged-product baseline passed. Root's independent final confirmation is
+pending at author submission; this report does not claim that it has run.
+
+## Frozen input and observer
+
+The dataset, SQL, bindings, sequence and budget were frozen before execution.
+The fixture has 10 Monitors and 99,000 CheckRuns: A has 90,000; B–J have 1,000
+each. Every 100th global ordinal is FAILED. Alice owns A–I and Bob owns J.
+
+| File | SHA-256 |
+| --- | --- |
+| `fixture.json` | `7ec00fd2734f5a21c739d10cc93634d6f9d621a4c225328bd5d73be3ab61ee4b` |
+| `fixtures.md` | `fdc3b5c8cc2031f60a3a1c894adf1c4003f028636e80b4370f150fbe4019dacf` |
+| `history.sql` | `28a76a06b4c6725dd412a9762423edfcebb635c48ababf7a981ba2ad4e1020d2` |
+| `seed.sql` | `76e01303ea456e8db0b6a241bd429f21dceac52ccca06d78853fd5689977ccf4` |
+| `HistoryQueryPlanTest.java` | `fb5660ea1860b868e6dd9eed0a8d80bf2d4f9eac817bfca3b4e4c08574b58c23` |
+
+Before the first gate, source review added an injected MonitorStore parameter to
+the static seed method so the Spring context and Flyway initialize first. This
+was a pre-gate observer correction, not an executed failure or repair. The four
+frozen data/SQL/budget files did not change. The opt-in test reuses the existing
+HistoryPaginationTest SQL inspector, TestDatabase and account setup helpers.
+Default CI does not materialize the capped dataset.
+
+## Actual baseline
+
+The sole author gate, from the branch root with the pinned toolchain, was:
+
+```sh
+mvn -B -ntp -f backend/pom.xml -Dtest=HistoryQueryPlanTest -De20.plan-proof=baseline test
+```
+
+It passed 1 test, 0 failures/errors/skips: 7.662 seconds wall time, 6.564 seconds
+Maven time. The exact invocation is in `invocations.jsonl`; `author-output.json`
+preserves the native Maven console and JUnit text losslessly. Ordinary Mockito
+agent warnings remain in that output. No formal gate failed or was retried.
+
+The test compared the SQL emitted by the real production history call with the
+frozen SQL before executing EXPLAIN. Scoped TRACE recorded Monitor A, Alice,
+FAILED and the native limit 21. Public page size remains 20; the extra row
+determines the existing continuation cursor.
+
+| Observation | Baseline result |
+| --- | --- |
+| CheckRun access | `check_runs_history_state_order_idx`, actual 21 rows, one loop |
+| CheckRun rows removed by filter / index recheck | 0 / 0 |
+| CheckRun shared buffer hits / reads | 24 / 0 |
+| Whole-plan shared buffer hits / reads | 25 / 0 |
+| Explicit Sort / CheckRun sequential scan | Neither |
+| Monitor owner check | 10-row table scan, 1 match, 9 removed |
+| Returned records | First 20 plus next 20; all domain fields and order exact, 40 distinct IDs |
+| Foreign-owner read | Bob reading A: 404 |
+| Migration repeat | V1–V8 validated; 0 migrations applied; data/indexes unchanged |
+| Cleanup | Owned `e20_plan` schema dropped; test context closed |
+
+`baseline-plan.json` contains the complete plan; `baseline-result.json` contains
+the SQL/bind trace, exact records, index definitions/sizes, migration checksums
+and cleanup result. Planning 0.348 ms and execution 0.087 ms are observations
+only; no timing threshold or performance improvement is claimed. No worker,
+web server, browser or outbound fixture was started.
+
+## Candidate decision and write cost
+
+New candidate indexes evaluated: **0**. New indexes/migrations: **0**. Production
+changes: **0**. The existing V5 index already supplies the required state filter
+and finish/id ordering with exactly the bounded 21-row fetch. The independent
+root EXPLAIN will confirm the same conditions; there is no author duplicate
+final plan or fabricated before/after index comparison.
+
+The existing unfiltered history index occupies 10,321,920 bytes in this seed;
+the state-filtered history index occupies 13,631,488 bytes. The former still
+serves ordering across states. Both already incur index storage and maintenance
+on inserts or indexed state/finish changes, including WAL. An additional FAILED
+partial or covering index would add maintenance/storage without reducing the
+already bounded row fetch. None was created or benchmarked. E20 therefore adds
+no index write cost or storage; it does not claim that existing indexes are free.
+
+## Budget, reuse and pending confirmation
+
+| Activity | Author actual | Root reserved, not yet run |
+| --- | --- | --- |
+| Unique fixed datasets | 1 | Same dataset, no variant |
+| Dataset materializations / ANALYZE commands / EXPLAIN calls | 1 / 1 / 1 | 1 / 1 / 1 |
+| New index candidates | 0 of maximum 3 | No new candidate planned |
+| Formal gates / failures / repairs | 1 / 0 / 0 | One combined affected gate |
+| Duplicate author final plans | 0 | — |
+| Load runs / parameter sweeps / automatic retries | 0 / 0 / 0 | 0 / 0 / 0 |
+
+The START-to-author range changes only the opt-in test, its schema allowlist,
+E20 evidence and TRACK guidance. Product source, migrations, dependencies,
+frontend and runtime/verification configuration remain unchanged. Prior root
+evidence for those unchanged paths is reused, not reported as a new run. No
+full Maven suite, frontend type/build/browser gate, E11 kill or E12 outbound
+suite was repeated. Read-only source/report inspection and documentation checks
+are not additional test gates; an attempted read of an absent prior E12 README
+returned no such file and had no effect on acceptance or artifacts.
+
+Root's reserved command is:
+
+```sh
+mvn -B -ntp -f backend/pom.xml -Dtest=HistoryQueryPlanTest,HistoryPaginationTest,HistoryIndexMigrationTest -De20.plan-proof=final test
+```
+
+It may re-materialize the identical fixed dataset once and run the existing
+history/tie and migration regressions. Root owns its result, final scope audit,
+tag and index. E24 and all later work remain unstarted.
