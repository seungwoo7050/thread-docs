## `fix(android): release focus before disabling storage controls`

diff --git a/SPEC_REVISION b/SPEC_REVISION
index 81ee29e..c839290 100644
--- a/SPEC_REVISION
+++ b/SPEC_REVISION
@@ -1 +1 @@
-ed7baa246ee947c6778e0f84751c9b91cec7abfe
+61280dd86ce88b6e431f408241c0998a275960aa
diff --git a/TRACK.md b/TRACK.md
index 3b61ebb..2a05222 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,7 +1,10 @@
 # Android Kotlin — Offline Item Tracker
 
 Independent orphan branch: `track/android-kotlin`.
-The fixed specification revision is recorded in `SPEC_REVISION` and every commit trailer.
+Active profile: `phase-1`; specification revision
+`61280dd86ce88b6e431f408241c0998a275960aa` is recorded in `SPEC_REVISION`.
+The profile covers M01–M10 → M14 → phase-1 M15; phase-2 work and remote push are excluded.
+Earlier commits and evidence retain their original revision; new commits include the profile.
 
 ## M01 boundary
 
@@ -92,6 +95,13 @@ has ten passes and the same M01 Compose focus failure. **M04 is BLOCKED after bo
 repairs; no progress tag or M05 work follows.** Its failed final verification and diagnostic
 limits are recorded in `verification/M04.md`. The repair did not change product code or inputs.
 
+The user subsequently authorized continuation after that recorded stop. Supplemental repair1
+(attempt4 overall, cumulative repair3) moves focus clearing before storage disables the
+controls, preserving success-only draft clearing and every test/fixture input. The six host
+tests, main's unchanged eleven-method Android suite, direct M03 seed and schema-2 external
+process restart now pass. No additional device run was performed by the repair agent.
+The earlier failures remain in `verification/M04.md`; main owns final commit acceptance/tagging.
+
 M04 reproduced M03's missing freshness and successful-refresh-time presentation without
 changing its working cached reads. The ordinary screen still reads Room before any network
 request. **Sync** remains the only refresh trigger; restoring transport does not itself send
diff --git a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
index 6fd9e49..09061c6 100644
--- a/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
+++ b/app/src/main/java/com/mobilesystemsevolution/kotlin/MainActivity.kt
@@ -86,6 +86,8 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
 
     fun change(action: suspend () -> Unit, after: () -> Unit = {}, isSync: Boolean = false) {
         if (busy) return
+        // Release focus before busy disables and removes the controls' focus targets.
+        focus.clearFocus()
         busy = true
         syncing = isSync
         scope.launch { accessStorage(action, after) }
@@ -119,7 +121,6 @@ internal fun ItemScreen(store: ItemStore, sync: ItemSync? = null) {
                         after = {
                             title = ""
                             editingId = null
-                            focus.clearFocus()
                         },
                     )
                 },
diff --git a/verification/M04.md b/verification/M04.md
index b9558df..a750658 100644
--- a/verification/M04.md
+++ b/verification/M04.md
@@ -377,3 +377,102 @@ and active network 125. No OS settings were changed. The direct diagnostic insta
 before that diagnostic; no restart-boundary test was performed under this lease.
 `diagnostic-lease-release.json` and raw device commands preserve cleanup. No M05 work,
 production change, retry framework or other new scope was added.
+
+## Supplemental repair 1 — attempt 4 overall, cumulative repair 3
+
+**PASS: six host tests, main's eleven Android methods, one actual direct M03 seed, and
+schema-2 external process restart. Earlier failed attempts and their counts are retained.**
+
+The user explicitly authorized continuation after receiving the BLOCKED report; main records
+this in `threads/RESUME-2026-08-28.md`. This is the same M04 from clean START
+`05d552ab9580692a40ea997e8c34db516bfb1df4`, with the original SPEC_REVISION and frozen inputs.
+Raw evidence is `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/android-kotlin/M04/supplemental-repair1/`
+(`E` below). `preflight.json`, `start-source.tar`, starting APKs and `prior-main-failure/`
+preserve the exact source and rejected main result before this correction.
+
+### Focus correction and unchanged scope
+
+The previous failure occurred in semantics `requestFocus` after the Beta readiness barrier.
+The app set `busy=true`, disabling its focused editor, before asynchronous Room/status work;
+focus clearing happened only in the later success callback. Pinned Compose 1.6.8 sources show
+that `focusable(enabled=false)` removes the focus target. The existing `clearFocus()` call
+now runs immediately before `busy=true`, while the focused controls are still attached.
+Its former success-callback call is removed; title/edit selection still clear only after
+successful storage work, and failed writes retain the draft and confirmed list.
+[FocusManager's public API](https://developer.android.com/reference/kotlin/androidx/compose/ui/focus/FocusManager)
+documents clearing the active path to the root. `diagnosis-before-change.md` and the pinned
+source archives retain the evidence and limits: the exact earlier internal tree mutation was
+not observed, and no library defect is claimed.
+
+This is the only execution-file change: MainActivity.kt has two added lines and one removed
+line. Every test, actual UI action, assertion, timeout, fixture, schema, build input and
+restart harness remains byte-identical to START. No waits or retries were added.
+
+### Commands and candidate identity
+
+`G` and its environment retain the definitions above. `E/commands.jsonl` records the exact
+agent commands, times, exits and logs; main's complete commands/results are in `main-final-01/`.
+
+| Invocation | Exit | Actual result |
+| --- | --- | --- |
+| Official UI/Foundation 1.6.8 source downloads, each `-01` / `-02` | 6 / 0 | Sandbox DNS failures, then approved downloads; no tests |
+| `G :app:testDebugUnitTest --rerun :app:assembleDebug :app:assembleDebugAndroidTest` | 0 | One build, 49.671s; all six host tests executed and passed; test APK unchanged |
+| `freeze-candidate-01` / `-02` | 1 / 0 | External snapshot script initially assumed optional local.properties existed; metadata corrected, no source/build/test change; both snapshots' source/APKs/XML/diff are identical |
+| Main `G :app:connectedDebugAndroidTest` | 0 | One suite: 11/11 pass, zero skips, 42.123s test time / 57s Gradle; existing APKs reused |
+| Main `direct-seed` | 1 | Setup failure: UTP had removed instrumentation; zero tests executed |
+| Main same-APK installs, then `direct-seed-02` | 0 | Both installs precede seeding/restart; unchanged M03 method passes once, `OK (1 test)`, 14.131s test time |
+| Main unchanged `canonical_restart.py --schema-version 2` | 0 | 18 recorded adb calls; actual PID 7346 → absent → 7429; exact native data and UI retained |
+| `receive-main-01` | 0 | All 32 frozen tracked files and both APKs match; copied/hashed 86 main evidence files and audited native DB/WAL copies; no new tests |
+
+The successful freeze is `fixed-built-02/manifest.json`; the first incomplete metadata
+snapshot is retained. `freeze-retry-audit.json` confirms no extra build or test execution.
+
+| Frozen artifact | SHA-256 |
+| --- | --- |
+| Manifest | `8b65be9c52a2ff325905824c425558ea3defd3089caabfa5e4f06de910a010ba` |
+| App APK | `f86e8e0b0875450055139e03d892dbf21c3471b00ca89b97c176c45b6c9d3ae6` |
+| Test APK, unchanged from START | `622a4047195a1e5d5e8e567a2a25d6e0d6d0822f1a4f6d5022575bfed8f8a231` |
+| Main Android XML | `b2f985b11ca4d5ef4cc6600b0b162478c4fd0c4bc0b38b64145f6d57db82542f` |
+
+### Final runtime evidence and cleanup
+
+Main's raw evidence is
+`/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/android-kotlin/M04/main-supplemental1-01/`.
+`main-final-inspection.json` verifies its XML, original method set, command results, source
+identity and copied-file hashes. M01 now reaches rename, all three completion changes and
+deletion. M04 logs all four original phases: failures retain both rows/time, and reconnection
+produces Remote revised/version2 and Beta/version1 at 1700000202000 with four attempts and
+three delivered requests. The failed-write draft/list and pending-HTTP local-read cases pass.
+
+The restart boundary has no install, clear, instrumentation or database write. Disposable
+copies of the actual SQLite/WAL show schema2, the five Item columns, exact Gamma/Alpha synced
+canonical rows and unchanged refresh metadata before/after. Both saved screenshots were
+visually checked. The main setup failure is preserved separately from the one actual seed;
+it is not counted as a passing test or hidden as a rerun.
+
+Main stopped its owned fixture PID14360 with SIGINT, tool session9199 exit0. Ten cleanup
+commands verify the app absent, port18080 free and network0/1/1 with active default network103.
+Expected empty app-PID and final port queries retain exit1. The repair agent made no device
+call, fixture launch, additional Android diagnostic or full-suite repetition. A coordinator
+status-list tool incidentally returned a peer's final report; it was not used and no peer
+files were opened (`boundary-note.json`). Only TRACK's M04 report and this ledger changed
+after runtime verification; the committed execution bytes remain the frozen candidate.
+No M05 work or progress tag is created by this repair.
+
+### Phase-1 transition before final commit
+
+The user paused finalization before any M04 commit and authorized the spec-only main commit
+`61280dd86ce88b6e431f408241c0998a275960aa`, now the active `phase-1` SPEC_REVISION. Main preserved
+the complete pending patch in `threads/history/profile-transition-2026-08-28/android-kotlin/`
+and adopted M03 at `f3591d527b40bd00e2a885c4a53d2508bae209fd` under
+`progress/phase-1/android-kotlin/M03`. Main rechecked unchanged M04 acceptance and approved
+reuse of the passing runtime evidence above. Finalization changes only revision/report
+metadata after that freeze; product, tests, fixtures, build inputs and both APKs remain
+identical to the tested candidate. No build, test or device operation is repeated.
+Attempt4 and cumulative repair3 remain historical counts; the supplemental repair was
+authorized before this transition. The current hard repair cap leaves no further repair
+for M04. Prior revisions, failures, commits and evidence are preserved without rewriting.
+New transition/hash/commit audit records are under
+`/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/android-kotlin/M04/finalization-01/`.
+The final commit uses contiguous `Thread: M04`, `Profile: phase-1` and the new revision
+trailers. No successor, phase-2 implementation or remote push is included.
