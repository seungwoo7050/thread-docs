## `feat(m09): resume durable uploads on foreground startup`

diff --git a/TRACK.md b/TRACK.md
index 9d44513..d1dc714 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -191,6 +191,36 @@ rendered controls and native database copies. Host tests separately cover editor
 ownership and disposed callbacks. See `verification/M08.md` for the unchanged-app
 baseline, raw commands and main's final Android verification status.
 
+## M09: foreground startup recovery after process death (phase-1)
+
+After reading the committed local snapshot, startup invokes the existing
+foreground sync only when persisted nonterminal intent exists. The same busy
+guard prevents a simultaneous manual drain, and a disposed opening effect cannot
+start recovery. Empty and terminal-only queues keep their prior cached startup
+behavior. New edits in the running screen still wait for explicit sync; there is
+no polling, connectivity listener, scheduler or background guarantee.
+
+The queue, immutable dispatched envelopes, ordered upload, identity replay,
+conflict policy and atomic acknowledgment/dequeue are unchanged. An offline
+startup attempt can mark its head dispatched before fetch fails; it retains the
+exact payload/hash/identity and last successful-refresh time. Ordinary cold
+launch after an interrupted request recovers from SQLite, not editor memory.
+The only native change is a DEBUG-gated fixed `death` Item prefix for the frozen
+test; mutation identity uses the existing debug input.
+
+The M09 fixture exposes a committed-response barrier with a maximum30000ms hold
+and responsive controls. Its external harness confirms a still-open, headerless
+response immediately before actual process death, then drops it only after the
+PID is absent. Fixed case A creates through the real UI; baseline A's approved
+prelaunch database input is explicitly not production-create evidence.
+
+Historical M05–M07 harnesses retain their earlier manual-first-Sync timing and
+must use their recorded prior artifacts. They are not silently adapted to M09.
+The current host suite covers those interactions. Main's M09 process-death,
+M08 real Activity lifecycle and native CRUD checks passed on the frozen candidate;
+prior M05–M07 core Android evidence was reused, not rerun. See
+`verification/M09.md` for the exact artifacts, counts and audit links.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 183ab5d..2faa6e9 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -58,7 +58,8 @@ test('M01 fixed sequence maps stable Item identity to the rendered list', async
 test('M02 startup reads saved Items; failed commits leave the draft and confirmed UI unchanged', async () => {
   const store = await openItemStore();
   const original = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  render(<App testIdentityPrefix="item" />);
+  // This is a local commit/rollback test; M09 startup may invoke its session.
+  render(<App testIdentityPrefix="item" createSync={() => ({initialized: false, identityPrefix: 'item', synchronize: async () => {}})} />);
   await saved();
   expect(screen.getByText('Alpha')).toBeTruthy();
   failNextSql(/^END/);
@@ -302,17 +303,21 @@ test('M05 UI reads durable pending work after restart and clears it after a fore
     expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
     expect(screen.queryByText('Beta')).toBeNull();
     expect(request).not.toHaveBeenCalled();
+    const pendingBeforeRestart = await store.readPending();
     view.unmount();
     closeDatabases();
     const reopened = await openItemStore();
     render(<App openStore={async () => reopened} createSync={createSync} />);
     await saved();
-    expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+    expect(screen.getByLabelText('Sync status: error')).toBeTruthy();
     expect(screen.getByLabelText('Pending uploads: 4')).toBeTruthy();
     expect(screen.getByText('Gamma')).toBeTruthy();
     expect(screen.getByRole('checkbox', {name: 'Mark Queued edit incomplete'}).props.accessibilityState.checked).toBe(true);
     expect(screen.queryByText('Beta')).toBeNull();
-    expect(request).not.toHaveBeenCalled();
+    expect(request).toHaveBeenCalledTimes(1); // Offline startup attempted the persisted head, without HTTP acceptance.
+    expect(await reopened.readPending()).toEqual(pendingBeforeRestart.map((operation, index) =>
+      index === 0 ? {...operation, dispatched: true} : operation));
+    expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
     offline = false;
     fireEvent.press(screen.getByLabelText('Synchronize'));
     await saved();
@@ -321,7 +326,7 @@ test('M05 UI reads durable pending work after restart and clears it after a fore
     expect(await reopened.read()).toEqual(final);
     expect(await reopened.readPending()).toEqual([]);
     expect(replies).toEqual([]);
-    expect(request).toHaveBeenCalledTimes(5);
+    expect(request).toHaveBeenCalledTimes(6);
   } finally {clock.mockRestore();}
 });
 
@@ -479,7 +484,9 @@ test('M08 a late Save completion cannot clear the remounted draft or dismiss its
   });
   const dismiss = jest.spyOn(Keyboard, 'dismiss');
   const pendingReads = jest.spyOn(store, 'readPending');
-  const props = {openStore: async () => store};
+  // Keep this late-Save/editor test independent of M09's startup transport.
+  const props = {openStore: async () => store,
+    createSync: () => ({initialized: false, identityPrefix: 'device', synchronize: async () => {}})};
   try {
     const first = render(<App {...props} />);
     await saved();
@@ -554,3 +561,73 @@ test('M08 a disposed opening effect cannot install a duplicate session or publis
   expect(await store.read()).toEqual([m08Input.seed]);
   expect(await store.readPending()).toEqual([]);
 });
+
+const m09Input: typeof import('../verification/M09-inputs.json') = require('../verification/M09-inputs.json');
+
+test.each(m09Input.cases)('M09 startup resumes persisted case $case without a Sync event and keeps cached UI busy until ACK', async scenario => {
+  // Component trigger coverage with real host SQLite. The external Android
+  // harness separately proves the actual server commit and process boundary.
+  const store = await openItemStore(undefined, () => scenario.clientMutationId);
+  if (scenario.case === 'B') {await store.replaceSnapshot(scenario.initialItems, 1700000000000);}
+  await store.mutate(scenario.case === 'A'
+    ? {type: 'create', title: 'Recovered create', now: 1700000599000}
+    : {type: 'rename', id: 'death-001', title: 'Recovered update', now: 1700000599000}, 'death');
+  if (scenario.case === 'B') {await store.prepareNext();}
+  const original = await store.read();
+  const pending = await store.readPending();
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.readPending()).toEqual(pending);
+  let release!: (reply: Awaited<ReturnType<JsonRequest>>) => void;
+  const held = new Promise<Awaited<ReturnType<JsonRequest>>>(resolve => {release = resolve;});
+  const request: JsonRequest = jest.fn(async (address, options) => {
+    if (options.method === 'GET') {
+      expect(address).toBe('http://fixed-m09/items');
+      expect(options.body).toBeUndefined();
+      return {status: 200, json: async () => ({items: [scenario.finalItem]})};
+    }
+    expect(address).toBe('http://fixed-m09' + scenario.path);
+    expect(options.method).toBe(scenario.method);
+    expect(JSON.parse(options.body!)).toEqual(scenario.wireBody);
+    return held;
+  });
+  render(<App openStore={async () => reopened}
+    createSync={() => new ForegroundSync(reopened, 'http://fixed-m09', request, 'unused-new-session', () => 1700000601000)} />);
+  await waitFor(() => expect(request).toHaveBeenCalledTimes(1));
+  expect(screen.getByLabelText('Sync status: refreshing')).toBeTruthy();
+  expect(screen.getByText(scenario.finalItem.title)).toBeTruthy();
+  expect(screen.getByLabelText('Pending uploads: 1')).toBeTruthy();
+  expect(screen.getByLabelText('Synchronize').props.accessibilityState.disabled).toBe(true);
+  expect(await reopened.read()).toEqual(original);
+  expect(await reopened.readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  const manual = screen.UNSAFE_getAllByType(Button).find(button => button.props.title === 'Synchronize')!.props.onPress;
+  await act(async () => {await manual();}); // The ref guard also blocks a queued handler.
+  expect(request).toHaveBeenCalledTimes(1);
+  await act(async () => {release({status: scenario.status, json: async () => ({item: scenario.finalItem})});});
+  await saved();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(screen.getByLabelText('Pending uploads: 0')).toBeTruthy();
+  expect(await reopened.read()).toEqual([scenario.finalItem]);
+  expect(await reopened.readPending()).toEqual([]);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000601000);
+  expect(request).toHaveBeenCalledTimes(2);
+});
+
+test('M09 a disposed database opening cannot start recovery for its still-pending intent', async () => {
+  const oldStore = await openItemStore('m09-disposed.db', () => 'm09-create-001');
+  await oldStore.mutate({type: 'create', title: 'Recovered create', now: 1700000599000}, 'death');
+  const pending = await oldStore.readPending();
+  let finishOldOpen!: (store: typeof oldStore) => void;
+  const opening = new Promise<typeof oldStore>(resolve => {finishOldOpen = resolve;});
+  const oldSync = jest.fn(() => ({initialized: false, identityPrefix: 'old', synchronize: jest.fn(async () => {})}));
+  const first = render(<App openStore={() => opening} createSync={oldSync} />);
+  first.unmount();
+  const currentStore = await openItemStore('m09-current.db');
+  render(<App openStore={async () => currentStore} />);
+  await saved();
+  await act(async () => {finishOldOpen(oldStore); await opening;});
+  expect(oldSync).not.toHaveBeenCalled();
+  expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(await oldStore.readPending()).toEqual(pending);
+});
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index 87655e4..a14bff0 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -22,6 +22,9 @@ class MainActivity : ReactActivity() {
                         putString("testIdentityPrefix", "crash")
                         putString("testMutationIdentity", "m06-create-001")
                     }
+                    if (intent.getBooleanExtra("m09FixedIdentity", false)) {
+                        putString("testIdentityPrefix", "death")
+                    }
                     intent.getStringExtra("m07MutationIdentity")?.let {
                         putString("testMutationIdentity", it)
                     }
diff --git a/src/App.tsx b/src/App.tsx
index d8c18ff..bd58808 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -59,6 +59,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
 
   useEffect(() => {
     let active = true;
+    let recoverPending = false;
     busyRef.current = true;
     setBusy(true);
     setError(null);
@@ -80,11 +81,18 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         setConflictCount(conflicts.length);
         setRefresh({status: 'stale'});
         setReady(true);
+        recoverPending = pending.some(operation => operation.terminalError === null);
       }
     }).catch(reason => {
       if (active) {setError(`Could not open local database: ${String(reason.message ?? reason)}`);}
     }).finally(() => {
-      if (active) {busyRef.current = false; setBusy(false);}
+      if (active) {
+        busyRef.current = false;
+        // Only the committed startup queue triggers recovery. Live edits remain
+        // explicit; the existing sync path owns busy/error/ACK completion.
+        if (recoverPending) {void synchronize();}
+        else {setBusy(false);}
+      }
     });
     return () => {active = false;};
   }, [openStore, openAttempt, createSync, testIdentityPrefix, testMutationIdentity, testRefreshClock]);
@@ -184,7 +192,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
           {refresh.status === 'fresh' ? 'Fresh at last successful refresh'
             : refresh.status === 'refreshing' ? 'Refreshing; showing saved Items'
               : refresh.status === 'error' ? 'Refresh error; saved Items may be stale'
-                : 'Stale; showing saved Items until explicit refresh'}
+                : 'Stale; showing saved Items'}
         </Text>
         <Text accessibilityLabel={`Last successful refresh: ${lastSuccessfulRefreshAt ?? 'never'}`}>
           Last successful refresh: {lastSuccessfulRefreshAt === null ? 'never' : new Date(lastSuccessfulRefreshAt).toISOString()}
@@ -199,7 +207,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
       </>}
       <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
-      <Text>Sync uploads saved edits in order, then refreshes Items.</Text>
+      <Text>Startup resumes saved uploads. Sync uploads edits in order, then refreshes Items.</Text>
       <TextInput
         accessibilityLabel={editingId === null ? 'New item title' : 'Edit item title'}
         placeholder="Item title"
diff --git a/verification/M09.md b/verification/M09.md
index 3b6f446..83c4bee 100644
--- a/verification/M09.md
+++ b/verification/M09.md
@@ -2,7 +2,7 @@
 
 - Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: verified/tagged M08 `f999421754533b3f112ad5b9a6aacac54b4b6b05`.
-- Attempt1; repairs0/2. Current status: **unchanged-app baseline accepted by main**; implementation and final verification pending.
+- Attempt1; repairs0/2. Current status: **main final Android verification PASS**, with host/build evidence audited; final history audit and tag remain main-owned.
 - Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/`.
 
 ## Frozen baseline
@@ -20,8 +20,26 @@ The single [baseline invocation](/private/tmp/mobile-systems-evolution-ed7baa2/e
 
 [Raw result and adjacent commands/HTTP/DB/UI evidence](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-android-01/result.json), [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-baseline-audit.json) and [cleanup readback](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/baseline-cleanup-readback.json) retain the complete proof. Host32023 and fixture32034 survived both app deaths; fixture exited0 and was directly absent, port18081 free, app absent, original network0/1/1 active restored. All59 execution files and the APK remained exact. The baseline-only lease was released; no repeat occurred.
 
-## Implementation and final verification plan
+## Implementation and host checks
 
-Main authorized only startup recovery through the existing foreground sync and the debug-only deterministic Item prefix needed for fixed A. No schema, hashing, acknowledgment, conflict algorithm, scheduler or new dependency is planned. Final device execution remains main-owned on a frozen candidate: M09 A/B, existing M08 actual Activity recreation and native CRUD.
+Baseline support/ledger is committed at `8144baee077eaa03c2b8028e04358b720e67dd24`; Git-parsed trailers and clean parent/history checks passed. Product changes were kept uncommitted for main's requested candidate freeze and official runs. Only `src/App.tsx` changes runtime behavior: after cached reads, eligible persisted intent invokes existing `synchronize()`, with its existing busy/disposal guards. Empty and terminal-only queues do not auto-sync; live edits remain explicit. The only native edit adds the DEBUG-gated `death` Item prefix. No schema, hashing, acknowledgment, conflict algorithm, scheduler or dependency changes.
+
+`host-jest-01` passed84/84 (5.940s). `host-typecheck-01` failed TS2345 because the new JSON-driven `test.each` had an ambiguous untyped callback overload. The exact failed source/log remain preserved. Adding only the fixture's compile-time `typeof import(...)` annotation changed no assertions or runtime behavior. `host-typecheck-02` passed (1.261s), and `host-jest-02` passed84/84 (5.587s) on the same corrected source used for the candidate. Four new tests comprise the real HTTP barrier case plus two persisted-startup cases and one disposed-opening case. The startup tests use actual host SQLite and the existing sync engine with a controlled transport, not a simulated Android death.
+
+The M05 component case now requires one failed offline startup attempt, exactly the head's dispatched-only transition, unchanged refresh time/envelopes, and the original four ordered accepts on explicit reconnect. Two earlier local-save/editor tests receive explicit sync doubles because pending startup now invokes transport; their original local assertions remain intact. Empty-queue, terminal collision, conflict, late-handler and lifecycle host coverage remains enabled. Exact commands, results and source snapshots are under the evidence root.
+
+At the candidate-freeze checkpoint, final Android execution was NOT_RUN and reserved for main. No owner fixed-device invocation occurred.
+
+`candidate-app-build-01` passed in12.264s, app-only. [Artifact/source preservation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/candidate-preservation.json) ties all59 nonledger files to both passing host checks and the build snapshot. App SHA256 is `76c12efda957b0100198cbc7f735368ca1c86e04d53c5c0c48a1c750d3e7b511`; only `assets/index.android.bundle` and `classes4.dex` differ from M08. The unchanged instrumentation APK remains `fcf6931c379bdbb2bcf6ec02c280e3f04b09b3de8068a77915850401856f3043`, with exact packaged M08 inputs; it was not rebuilt. [Candidate manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M09/candidate-manifest.json) records source/APK hashes, the uncommitted candidate at baseline HEAD, and exact main M09/M08/native-CRUD commands. Main requested product commits only after its official result.
 
 Historical M05/M06/M07 external harnesses remain unchanged. Their explicit-first-Sync timing predates M09: offline startup now may mark the head dispatched before a failed fetch; online startup may complete a replay before a manual tap. Current host regressions must cover that interaction without disabling startup recovery. Prior independently verified core Android evidence is reused only for unchanged storage/sync/protocol behavior; it is not reported as a new M09 execution.
+
+## Main final verification — PASS
+
+Main checked all60 frozen files and both APKs listed above, and audited the passing host84/typecheck/build snapshots. The official device runs were:
+
+- **M09 A/B PASS:** 71.135s,217 adb commands,11 native databases. A21981→absent→22501 recovered production UI create with applied1/duplicate0/v1; B22761→absent→23107 replayed its original base1 identity/hash with applied1/duplicate1/v2. Both ended pending0/t1700000600000. B's open, headerless commit→death→drop boundary completed within450ms; cold startup used no Sync tap or state injection. [Process-recovery audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-process-recovery-audit.json).
+- **M08 lifecycle PASS:** 17.954s,34 adb commands,6 native databases. PID23415/Application88250467/ReactContext136876754 remained stable; Activity46567430→128597992. Draft/selection survived; one later Save produced pending1 with HTTP0. [Lifecycle audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-lifecycle-regression-audit.json).
+- **Native CRUD PASS:** one JUnit test,25.919s. [CRUD audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-native-crud-audit.json).
+
+Fixtures48875/50064 exited0 and were directly absent; port18081 free, app absent, network0/1/1 active125 restored. [Cleanup audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M09/main-cleanup-audit.json). Initial readiness refusals and read-only audit corrections remain in raw records. No final candidate rejection, repair, owner rebuild or device repeat occurred; repairs remain0/2. This closeout updates only TRACK/ledger metadata before committing the exact tested candidate; main owns the final Git audit/tag.
