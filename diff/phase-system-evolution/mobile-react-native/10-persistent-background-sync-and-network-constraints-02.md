## `M10: persist bounded Android background uploads`

diff --git a/TRACK.md b/TRACK.md
index d1dc714..a0d84b3 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -221,6 +221,37 @@ M08 real Activity lifecycle and native CRUD checks passed on the frozen candidat
 prior M05–M07 core Android evidence was reused, not rerun. See
 `verification/M09.md` for the exact artifacts, counts and audit links.
 
+## M10: bounded Android background uploads (phase-1)
+
+Committed UI mutations now register unique, disk-backed WorkManager2.9.1 work
+with a CONNECTED constraint and exponential10s/20s retry delays. A small native
+bridge starts the existing JavaScript upload loop through a headless task.
+Checked SharedPreferences commits reserve at most three HTTP attempts per cycle;
+cycle and invocation tokens reject stale completion. Explicit JavaScript outcomes,
+not task-finish callbacks, determine success/retry/failure. Cancellation aborts
+and settles active HTTP before worker completion. Deferred work remains visible
+and requires the explicit Synchronize action to start another cycle.
+
+Startup preserves an owned cycle/count, recovering interrupted registration
+without resetting its allowance. Unowned persisted queues retain M09 foreground
+recovery. Successful late UI mutations still register work after their screen
+unmounts. UI and headless wrappers share only the production WebSQL connection
+and initialization; each retains its own identity source. The existing upload
+loop, dispatched envelopes and conflict policy remain in use without schema
+changes. Background uploads omit GET and never advance the refresh timestamp.
+An accepted canonical Item is promoted atomically with ACK/dequeue only when no
+later same-Item intent shadows it and the cached version permits it.
+
+The DEBUG clock input holds delayed callbacks for controlled verification while
+real WorkDatabase/SystemJobScheduler perform scheduling. Root verified actual
+same-UID process loss, OS-created replacement processes, persisted retry timing,
+exactly three HTTP attempts, and terminal deferred UI without a fourth request.
+M08 lifecycle and unchanged M01 CRUD passed under actual offline connectivity,
+with production scheduling enabled. Prior M05–M09 device evidence is retained,
+not rerun; current host regressions cover those shared semantics. No phase-2
+feature or push integration is included. See `verification/M10.md` and its frozen
+manifest for the separate, budgeted A/B commands and complete raw evidence.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
@@ -256,8 +287,8 @@ instrumentation APK use Android's generated debug signing key. This is not a
 production signing or release validation guarantee.
 
 The pinned [React Native 0.76 template](https://github.com/react-native-community/template/tree/0.76-stable/template)
-provides the platform bootstrap conventions. Legacy architecture is sufficient for
-this minimal screen; there are no app-specific native extensions. UI tests use
+provides the platform bootstrap conventions. Legacy architecture remains in use;
+M10 adds only its required WorkManager worker and headless upload bridge. UI tests use
 AndroidX UI Automator to operate the actual React Native Android controls. Host tests
 independently cover the fixed domain and rendered-list mutation sequence.
 
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 2faa6e9..8dbc283 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -1,16 +1,22 @@
 import React from 'react';
-import {Button, Keyboard} from 'react-native';
+import {Button, DeviceEventEmitter, Keyboard, NativeModules} from 'react-native';
 import {act, fireEvent, render, screen, waitFor} from '@testing-library/react-native';
 import RootApp, {createEditorMemory} from '../src/App';
 import {openItemStore} from '../src/itemStore';
 import {closeDatabases, failNextSql} from './sqliteNative';
 import {ForegroundSync, JsonRequest} from '../src/sync';
+import {BackgroundState, serializeSync} from '../src/backgroundSync';
 
 const saved = () => waitFor(() => expect(screen.getByLabelText('Local storage ready')).toBeTruthy());
 let editorMemory: ReturnType<typeof createEditorMemory>;
 beforeEach(() => {editorMemory = createEditorMemory();});
 // Each test owns one JS-session editor; remounts within it retain that owner.
-const App = (props: React.ComponentProps<typeof RootApp>) => <RootApp editorMemory={editorMemory} {...props} />;
+// Inject a per-test SQLite opener rather than carry the production process
+// singleton across independent database resets. Its owner is covered below
+// the bridge in sync.test.ts; all existing lifecycle assertions stay intact.
+const openForScreen = () => openItemStore();
+const App = (props: React.ComponentProps<typeof RootApp>) =>
+  <RootApp editorMemory={editorMemory} openStore={openForScreen} {...props} />;
 
 test('M01 fixed sequence maps stable Item identity to the rendered list', async () => {
   let clock = 1700000000000;
@@ -631,3 +637,137 @@ test('M09 a disposed database opening cannot start recovery for its still-pendin
   expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
   expect(await oldStore.readPending()).toEqual(pending);
 });
+
+test('M10 an owned active queue uses registration recovery instead of M09 foreground HTTP on mount', async () => {
+  const store = await openItemStore(undefined, () => 'unit-owned');
+  await store.mutate({type: 'create', title: 'Owned queue', now: 1700000600000}, 'unit-owned');
+  const pending = await store.readPending();
+  const state: BackgroundState = {cycleId: 'existing-cycle', attempts: 1, status: 'active'};
+  NativeModules.BackgroundSync.getState.mockResolvedValue(state);
+  NativeModules.BackgroundSync.schedule.mockResolvedValue(state);
+  const synchronize = jest.fn(async () => {});
+  render(<App openStore={async () => store}
+    createSync={() => ({initialized: false, identityPrefix: 'unused', synchronize})} />);
+  await saved();
+  expect(screen.getByLabelText('Background sync: active')).toBeTruthy();
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(NativeModules.BackgroundSync.schedule).toHaveBeenCalledTimes(1);
+  expect(NativeModules.BackgroundSync.prepareManual).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.reserve).not.toHaveBeenCalled();
+  expect(synchronize).not.toHaveBeenCalled();
+  expect(await store.readPending()).toEqual(pending);
+});
+
+test('M10 deferred queue remains visible across ordinary remount with no new automatic cycle or foreground request', async () => {
+  const store = await openItemStore(undefined, () => 'unit-deferred');
+  await store.mutate({type: 'create', title: 'Deferred queue', now: 1700000600000}, 'unit-deferred');
+  await store.prepareNext();
+  const pending = await store.readPending();
+  const state: BackgroundState = {cycleId: 'exhausted-cycle', attempts: 3, status: 'deferred'};
+  NativeModules.BackgroundSync.getState.mockResolvedValue(state);
+  const synchronize = jest.fn(async () => {});
+  const props = {openStore: async () => store,
+    createSync: () => ({initialized: false, identityPrefix: 'unused', synchronize})};
+  const first = render(<App {...props} />);
+  await saved();
+  first.unmount();
+  render(<App {...props} />);
+  await saved();
+  expect(screen.getByLabelText('Background sync: deferred')).toBeTruthy();
+  expect(screen.getByText(/deferred after 3 reserved attempts/)).toBeTruthy();
+  expect(screen.getByLabelText('Pending uploads: 1')).toBeTruthy();
+  expect(NativeModules.BackgroundSync.schedule).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.prepareManual).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.reserve).not.toHaveBeenCalled();
+  expect(synchronize).not.toHaveBeenCalled();
+  expect(await store.readPending()).toEqual(pending);
+});
+
+test('M10 a committed edit waits for durable registration only, not worker execution', async () => {
+  let registered!: (state: BackgroundState) => void;
+  const registration = new Promise<BackgroundState>(resolve => {registered = resolve;});
+  NativeModules.BackgroundSync.schedule.mockReturnValue(registration);
+  const store = await openItemStore();
+  render(<App openStore={async () => store} testIdentityPrefix="unit-registration" />);
+  await saved();
+  fireEvent.changeText(screen.getByLabelText('New item title'), 'Registered edit');
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await waitFor(() => expect(NativeModules.BackgroundSync.schedule).toHaveBeenCalledTimes(1));
+  expect(screen.getByLabelText('Local storage busy')).toBeTruthy();
+  expect(await store.readPending()).toHaveLength(1);
+  await act(async () => {registered({cycleId: 'registered-cycle', attempts: 0, status: 'active'});});
+  await saved();
+  expect(screen.getByLabelText('Background sync: active')).toBeTruthy();
+  expect(NativeModules.BackgroundSync.reserve).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.complete).not.toHaveBeenCalled();
+});
+
+test('M10 a scheduler error after COMMIT retains the edit and reports scheduling failure, not an unsaved change', async () => {
+  NativeModules.BackgroundSync.schedule.mockRejectedValue(new Error('Registration failed'));
+  const store = await openItemStore();
+  render(<App openStore={async () => store} />);
+  await saved();
+  fireEvent.changeText(screen.getByLabelText('New item title'), 'Durable edit');
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await saved();
+  expect(screen.getByText('Background scheduling failed; saved changes are retained.')).toBeTruthy();
+  expect(screen.queryByText('Change not saved')).toBeNull();
+  expect((await store.read())[0].title).toBe('Durable edit');
+  expect(await store.readPending()).toHaveLength(1);
+  expect(screen.getByLabelText('New item title').props.value).toBe('');
+});
+
+test('M10 a committed late Save still registers work while suppressing old editor callbacks and status reads', async () => {
+  const store = await openItemStore();
+  let release!: () => void;
+  const response = new Promise<void>(resolve => {release = resolve;});
+  let committed!: () => void;
+  const didCommit = new Promise<void>(resolve => {committed = resolve;});
+  const original = store.mutate.bind(store);
+  jest.spyOn(store, 'mutate').mockImplementation(async (...args) => {
+    const rows = await original(...args); committed(); await response; return rows;
+  });
+  const reads = jest.spyOn(store, 'readPending');
+  const first = render(<App openStore={async () => store} />);
+  await saved();
+  fireEvent.changeText(screen.getByLabelText('New item title'), 'Late durable edit');
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await act(async () => {await didCommit;});
+  first.unmount();
+  const readsAtUnmount = reads.mock.calls.length;
+  await act(async () => {release(); await response; await serializeSync(async () => {});});
+  expect(NativeModules.BackgroundSync.schedule).toHaveBeenCalledTimes(1);
+  expect(reads).toHaveBeenCalledTimes(readsAtUnmount);
+  expect(editorMemory.current).toEqual({editingId: null, draft: 'Late durable edit'});
+});
+
+test('M10 background listener cleanup and stale callback guards leave a remounted draft untouched', async () => {
+  const oldStore = await openItemStore('unit-old-root.db');
+  await oldStore.replaceSnapshot([m08Input.seed]);
+  const oldRead = jest.spyOn(oldStore, 'read');
+  const listen = jest.spyOn(DeviceEventEmitter, 'addListener');
+  const first = render(<App openStore={async () => oldStore} />);
+  await saved();
+  const index = listen.mock.calls.findIndex(args => args[0] === 'BackgroundSyncChanged');
+  const callback = listen.mock.calls[index][1];
+  const remove = jest.spyOn(listen.mock.results[index].value, 'remove');
+  first.unmount();
+  expect(remove).toHaveBeenCalledTimes(1);
+  const readsAtUnmount = oldRead.mock.calls.length;
+  const currentStore = await openItemStore('unit-current-root.db');
+  await currentStore.replaceSnapshot([m08Input.seed]);
+  const current = render(<App openStore={async () => currentStore} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Current draft');
+  await act(async () => {
+    callback(undefined);
+    DeviceEventEmitter.emit('BackgroundSyncChanged');
+    await serializeSync(async () => {});
+  });
+  expect(oldRead).toHaveBeenCalledTimes(readsAtUnmount);
+  expect(screen.getByLabelText('Edit item title').props.value).toBe('Current draft');
+  expect(editorMemory.current).toEqual({editingId: 'ui-001', draft: 'Current draft'});
+  current.unmount();
+  listen.mockRestore();
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index d89ab70..a6725a9 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -272,7 +272,7 @@ const lastAcknowledgement = () => {
   return value === null ? null : JSON.parse(String(value));
 };
 
-test.each([/^UPDATE sync_metadata SET last_acknowledgement/, /^DELETE FROM pending_mutations/, /^END/])(
+test.each([/^UPDATE sync_metadata SET last_acknowledgement/, /^DELETE FROM pending_mutations/, /^UPDATE items SET title =/, /^END/])(
   'M06 acknowledgment failure at %s rolls back result recording and dequeue together', async failAt => {
     const store = await openItemStore(undefined, () => m06.clientMutationId);
     const local = await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
@@ -287,11 +287,15 @@ test.each([/^UPDATE sync_metadata SET last_acknowledgement/, /^DELETE FROM pendi
     expect(await reopened.readPending()).toEqual([sent]);
     expect(await reopened.read()).toEqual(local);
     expect(lastAcknowledgement()).toBeNull();
+    expect(connection().prepare('SELECT COUNT(*) AS count FROM remote_versions').get()?.count).toBe(0);
     await reopened.acknowledge(sent!, {item: m06.canonicalItem});
     expect(await reopened.readPending()).toEqual([]);
+    expect(await reopened.read()).toEqual([m06.canonicalItem]);
     expect(lastAcknowledgement()).toEqual(m06.acknowledgement);
     closeDatabases();
-    expect(await (await openItemStore()).readPending()).toEqual([]);
+    const recovered = await openItemStore();
+    expect(await recovered.readPending()).toEqual([]);
+    expect(await recovered.read()).toEqual([m06.canonicalItem]);
     expect(lastAcknowledgement()).toEqual(m06.acknowledgement);
     console.info('M06_ATOMIC_ACK', JSON.stringify({failure: String(failAt), acknowledgement: lastAcknowledgement()}));
   });
@@ -432,19 +436,75 @@ test.each([/^INSERT INTO mutation_conflicts/, /^DELETE FROM pending_mutations/,
 test('M07 an old snapshot uses observed ACK fields, never optimistic fields masquerading as canonical', async () => {
   const input = m07.successorRegression;
   const store = await openItemStore(undefined, () => input.firstIdentity);
-  await store.replaceSnapshot([m07.seed]);
+  await store.replaceSnapshot([m07.seed], 1700000500000);
   const optimistic = await store.mutate({type: 'rename', id: m07.seed.id, title: 'Own predecessor', now: m07.seed.updatedAt});
   await store.acknowledge((await store.prepareNext())!, {item: input.firstResult});
-  expect(await store.read()).toEqual(optimistic); // The receipt is separate until reconciliation.
+  expect(await store.read()).toEqual([input.firstResult]); // M10 promotes a lone ACK without needing GET.
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000500000);
   expect(optimistic).not.toEqual([input.firstResult]);
   closeDatabases();
   const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual([input.firstResult]);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000500000);
   await reopened.replaceSnapshot([m07.seed]);
   expect(await reopened.read()).toEqual([input.firstResult]);
   expect(JSON.parse(String(connection().prepare('SELECT canonical_item FROM remote_versions').get()?.canonical_item)))
     .toEqual(input.firstResult);
 });
 
+test('M10 ACK promotion protects a later DELETE and its original identity', async () => {
+  let identity = 0;
+  const store = await openItemStore(undefined, () => `promotion-delete-${++identity}`);
+  await store.mutate({type: 'create', title: 'Temporary', now: 100}, 'promotion');
+  await store.mutate({type: 'delete', id: 'promotion-001'});
+  const first = (await store.prepareNext())!;
+  const later = (await store.readPending())[1];
+  await store.acknowledge(first, {item: {id: 'promotion-001', title: 'Temporary', completed: false, version: 1, updatedAt: 200}});
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual([]);
+  expect(await reopened.readPending()).toEqual([{...later, payload: {baseVersion: 1},
+    payloadHash: mutationHash('DELETE', '/items/promotion-001', {baseVersion: 1})}]);
+});
+
+test('M10 an unrelated pending Item does not prevent canonical ACK promotion', async () => {
+  let identity = 0;
+  const store = await openItemStore(undefined, () => `promotion-other-${++identity}`);
+  await store.mutate({type: 'create', title: 'First', now: 100}, 'promotion');
+  const optimistic = await store.mutate({type: 'create', title: 'Second', now: 101}, 'promotion');
+  const first = (await store.prepareNext())!;
+  const later = (await store.readPending())[1];
+  const canonical = {id: 'promotion-001', title: 'First', completed: false, version: 1, updatedAt: 200};
+  await store.acknowledge(first, {item: canonical});
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual([canonical, optimistic[1]]);
+  expect(await reopened.readPending()).toEqual([later]);
+});
+
+test.each(['newer live version', 'equal-version tombstone'] as const)(
+  'M10 a delayed ACK cannot overwrite an observed %s', async kind => {
+    let identity = 0;
+    const store = await openItemStore(undefined, () => `promotion-stale-${++identity}`);
+    await store.replaceSnapshot([m07.seed]);
+    await store.mutate({type: 'rename', id: m07.seed.id, title: 'First intent', now: 100});
+    await store.mutate({type: 'rename', id: m07.seed.id, title: 'Later intent', now: 101});
+    const rejected = (await store.prepareNext())!;
+    const winner = {...m07.seed, title: 'Observed winner', version: 3, updatedAt: 300};
+    const deleted = {id: m07.seed.id, version: 2, updatedAt: 300, deleted: true as const};
+    await store.rejectVersion(rejected, kind === 'newer live version' ? winner : null,
+      kind === 'equal-version tombstone' ? deleted : null);
+    const canonicalBefore = connection().prepare('SELECT * FROM remote_versions').get();
+    const next = (await store.prepareNext())!;
+    await store.acknowledge(next, {item: {...m07.seed, title: 'Old accepted value', version: 2, updatedAt: 200}});
+    closeDatabases();
+    const reopened = await openItemStore();
+    expect(await reopened.read()).toEqual(kind === 'newer live version' ? [winner] : []);
+    expect(connection().prepare('SELECT * FROM remote_versions').get()).toEqual(canonicalBefore);
+    expect(await reopened.readPending()).toEqual([]);
+    expect(lastAcknowledgement().clientMutationId).toBe(next.clientMutationId);
+  });
+
 test('M07 a successful DELETE receipt prevents resurrection before its timestamped tombstone arrives', async () => {
   const store = await openItemStore(undefined, () => m07.cases[1].clientMutationId);
   await store.replaceSnapshot([m07.seed]);
diff --git a/__tests__/sqlite.setup.ts b/__tests__/sqlite.setup.ts
index 0b903b2..2995593 100644
--- a/__tests__/sqlite.setup.ts
+++ b/__tests__/sqlite.setup.ts
@@ -9,5 +9,20 @@ jest.mock('react-native/Libraries/Utilities/Platform', () => ({
 }));
 
 NativeModules.RNSqlite2 = nativeSqlite;
+// Platform scheduling is not an in-process host test substitute. Existing
+// foreground cases use an unowned bridge; focused M10 tests inject owned state.
+const unowned = {cycleId: null, attempts: 0, status: 'none'};
+NativeModules.BackgroundSync = {
+  getState: jest.fn(), schedule: jest.fn(), prepareManual: jest.fn(),
+  isActive: jest.fn(), reserve: jest.fn(), requestFinished: jest.fn(), complete: jest.fn(),
+};
+beforeEach(() => {
+  for (const method of ['getState', 'schedule', 'prepareManual']) {
+    NativeModules.BackgroundSync[method].mockReset().mockResolvedValue(unowned);
+  }
+  for (const method of ['isActive', 'reserve', 'requestFinished', 'complete']) {
+    NativeModules.BackgroundSync[method].mockReset().mockResolvedValue(false);
+  }
+});
 beforeEach(resetSqlite);
 afterEach(cleanupSqlite);
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index b8cd6e8..47af3e8 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -1,7 +1,10 @@
 /// <reference types="node" />
 import {request as httpRequest, Server} from 'node:http';
-import {ForegroundSync, JsonRequest} from '../src/sync';
-import {MutationConflict, openItemStore, PendingMutation} from '../src/itemStore';
+import {DeviceEventEmitter} from 'react-native';
+import SQLite from 'react-native-sqlite-2';
+import {FIXTURE_URL, ForegroundSync, JsonRequest} from '../src/sync';
+import {BackgroundBridge, BackgroundState, runBackgroundTask, serializeSync} from '../src/backgroundSync';
+import {MutationConflict, openItemStore, openRuntimeItemStore, PendingMutation} from '../src/itemStore';
 import {mutationHash, mutationTarget} from '../src/mutationProtocol';
 import {Item} from '../src/items';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
@@ -33,6 +36,21 @@ const request: JsonRequest = (address, options) => new Promise((resolve, reject)
   outgoing.end(options.body);
 });
 
+// Bridge contract tests below exercise one request at a time. Native JUnit and
+// the separately budgeted Android cases verify disk allowance and OS dispatch.
+function backgroundContract(attempts = 0): jest.Mocked<BackgroundBridge> {
+  const state: BackgroundState = {cycleId: 'unit-cycle', attempts, status: 'active'};
+  return {
+    getState: jest.fn(async () => ({...state})),
+    schedule: jest.fn(async () => ({...state})),
+    prepareManual: jest.fn(async () => ({...state})),
+    isActive: jest.fn(async (_token: string): Promise<boolean> => true),
+    reserve: jest.fn(async (_token: string) => state.attempts < 3 ? (++state.attempts > 0) : false),
+    requestFinished: jest.fn(async (_token: string) => {}),
+    complete: jest.fn(async (_token: string, _outcome: 'success' | 'retry' | 'failure'): Promise<boolean> => true),
+  };
+}
+
 beforeAll(() => new Promise<void>((resolve, reject) => {
   fixture.server.once('error', reject);
   fixture.server.listen(18081, '127.0.0.1', resolve);
@@ -101,7 +119,14 @@ test('M03 snapshot replacement commits atomically and a failed local INSERT does
   await expect(sync.synchronize()).rejects.toThrow('injected persistence failure');
   expect(sync.initialized).toBe(false);
   closeDatabases();
-  expect(await (await openItemStore()).read()).toEqual(original);
+  const reopened = await openItemStore();
+  // M10 promotes the preceding accepted CREATE in its ACK transaction. The
+  // later failed snapshot transaction still must not publish either seed Item.
+  expect(await reopened.read()).toEqual([{...original[0], updatedAt: 1700000100000}]);
+  expect(await reopened.readPending()).toEqual([]);
+  expect(await reopened.readLastSuccessfulRefresh()).toBeNull();
+  expect(JSON.parse(String(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement)))
+    .toMatchObject({status: 201, result: {item: {...original[0], updatedAt: 1700000100000}}});
 });
 
 test('M03 distinct production sessions do not use the fixed test identity namespace', async () => {
@@ -655,3 +680,177 @@ test('M07 schema4 keeps original unversioned intent blocked and never infers can
     closeDatabases();
   }
 });
+
+test('M10 background drain commits one canonical ACK without GET or refresh time, then acknowledges before releasing the runtime lock', async () => {
+  const store = await openItemStore(undefined, () => 'unit-background-create');
+  await store.replaceSnapshot([], 1700000100123);
+  await store.mutate({type: 'create', title: 'Background unit', now: 1700000600000}, 'unit-bg');
+  const bridge = backgroundContract();
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn((address, options) =>
+    request(address.replace(FIXTURE_URL, url), options));
+  let entered!: () => void;
+  const completing = new Promise<void>(resolve => {entered = resolve;});
+  let release!: () => void;
+  const completion = new Promise<void>(resolve => {release = resolve;});
+  bridge.complete.mockImplementation(async () => {entered(); await completion; return true;});
+  const running = runBackgroundTask({token: 'unit-token'}, bridge, async () => store, transport);
+  try {
+    await completing;
+    const canonical = {id: 'unit-bg-001', title: 'Background unit', completed: false, version: 1, updatedAt: 1700000100000};
+    expect(await store.read()).toEqual([canonical]);
+    expect(await store.readPending()).toEqual([]);
+    expect(await store.readLastSuccessfulRefresh()).toBe(1700000100123);
+    expect(bridge.complete).toHaveBeenCalledWith('unit-token', 'success');
+    expect(bridge.reserve).toHaveBeenCalledTimes(1);
+    expect(bridge.requestFinished).toHaveBeenCalledTimes(1);
+    expect(transport).toHaveBeenCalledTimes(1);
+    expect(fixture.state().requests.map(event => [event.method, event.status])).toEqual([['POST', 201]]);
+    expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 0});
+    const laterMutation = jest.fn(async () => {});
+    const later = serializeSync(laterMutation);
+    await Promise.resolve();
+    expect(laterMutation).not.toHaveBeenCalled();
+    release();
+    await running;
+    await later;
+    expect(laterMutation).toHaveBeenCalledTimes(1);
+    closeDatabases();
+    const reopened = await openItemStore();
+    expect(await reopened.read()).toEqual([canonical]);
+    expect(await reopened.readPending()).toEqual([]);
+    expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000100123);
+  } finally {release(); await running;}
+});
+
+test.each([503, 409])('M10 one background HTTP %s reports its explicit result after transport settlement', async status => {
+  const store = await openItemStore(undefined, () => 'unit-background-rejected');
+  await store.mutate({type: 'create', title: 'Rejected unit', now: 1700000600000}, 'unit-reject');
+  const pending = (await store.readPending())[0];
+  const bridge = backgroundContract();
+  const events: string[] = [];
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn(async (_address: string, _options: Parameters<JsonRequest>[1]) => ({status,
+    json: async () => {events.push('body'); return {error: status === 409 ? 'identity_conflict' : 'Temporary unit failure'};}}));
+  bridge.requestFinished.mockImplementation(async () => {events.push('settled');});
+  bridge.complete.mockImplementation(async (_token, outcome) => {events.push(outcome); return true;});
+  await runBackgroundTask({token: 'unit-token'}, bridge, async () => store, transport);
+  expect(events).toEqual(['body', 'settled', status === 409 ? 'failure' : 'retry']);
+  expect(transport).toHaveBeenCalledTimes(1);
+  expect(bridge.reserve).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toEqual([{...pending, dispatched: true,
+    terminalError: status === 409 ? 'identity_conflict' : null}]);
+  expect(await store.readLastSuccessfulRefresh()).toBeNull();
+});
+
+test.each(['response', 'body'] as const)('M10 cancellation aborts the matching %s and waits for settlement without ACK or stale completion', async phase => {
+  const store = await openItemStore(undefined, () => 'unit-background-cancel');
+  await store.mutate({type: 'create', title: 'Cancelled unit', now: 1700000600000}, 'unit-cancel');
+  const pending = await store.readPending();
+  const bridge = backgroundContract();
+  let signal!: AbortSignal;
+  let started!: () => void;
+  const inFlight = new Promise<void>(resolve => {started = resolve;});
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn(async (_address, options) => {
+    signal = options.signal!;
+    const held = () => new Promise<never>((_resolve, reject) => {
+      signal.addEventListener('abort', () => reject(new Error('Aborted unit request')), {once: true});
+      started();
+    });
+    if (phase === 'response') {return held();}
+    return {status: 201, json: held};
+  });
+  const running = runBackgroundTask({token: 'unit-token'}, bridge, async () => store, transport);
+  await inFlight;
+  expect(bridge.requestFinished).not.toHaveBeenCalled();
+  DeviceEventEmitter.emit('BackgroundSyncCancelled', {token: 'older-token'});
+  expect(signal.aborted).toBe(false);
+  DeviceEventEmitter.emit('BackgroundSyncCancelled', {token: 'unit-token'});
+  await running;
+  expect(signal.aborted).toBe(true);
+  expect(bridge.requestFinished).toHaveBeenCalledTimes(1);
+  expect(bridge.complete).not.toHaveBeenCalled();
+  expect(transport).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  expect(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement).toBeNull();
+});
+
+test('M10 stale token does not open SQLite or ask for HTTP allowance', async () => {
+  const bridge = backgroundContract();
+  bridge.isActive.mockResolvedValue(false);
+  const open = jest.fn(openItemStore);
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn();
+  await runBackgroundTask({token: 'stale-token'}, bridge, open, transport);
+  expect(open).not.toHaveBeenCalled();
+  expect(bridge.reserve).not.toHaveBeenCalled();
+  expect(bridge.complete).not.toHaveBeenCalled();
+  expect(transport).not.toHaveBeenCalled();
+});
+
+test('M10 exhausted reservation cannot issue HTTP even when a worker reaches JS', async () => {
+  const store = await openItemStore(undefined, () => 'unit-exhausted');
+  await store.mutate({type: 'create', title: 'Retained unit', now: 1700000600000}, 'unit-limit');
+  const pending = await store.readPending();
+  const bridge = backgroundContract(3);
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn();
+  await runBackgroundTask({token: 'unit-token'}, bridge, async () => store, transport);
+  expect(transport).not.toHaveBeenCalled();
+  expect(bridge.requestFinished).not.toHaveBeenCalled();
+  expect(bridge.complete).toHaveBeenCalledWith('unit-token', 'retry');
+  expect(await store.readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  // BackgroundCycleTest independently checks that native retry at count3 is
+  // terminal/deferred, not a fourth WorkManager retry reservation.
+});
+
+test('M10 rejected reservation acknowledgment is settled before reporting failure without sending HTTP', async () => {
+  const store = await openItemStore(undefined, () => 'unit-reservation-error');
+  await store.mutate({type: 'create', title: 'Reservation unit', now: 1700000600000}, 'unit-reservation');
+  const bridge = backgroundContract();
+  bridge.reserve.mockRejectedValue(new Error('Reservation bridge reply failed'));
+  const events: string[] = [];
+  bridge.requestFinished.mockImplementation(async () => {events.push('settled');});
+  bridge.complete.mockImplementation(async (_token, outcome) => {events.push(outcome); return true;});
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn();
+  await runBackgroundTask({token: 'unit-token'}, bridge, async () => store, transport);
+  expect(events).toEqual(['settled', 'failure']);
+  expect(transport).not.toHaveBeenCalled();
+  expect(await store.readPending()).toHaveLength(1);
+});
+
+test('M10 production callers share one SQL queue, retain per-caller identity and allow ACK while an old callback is delayed', async () => {
+  const openingSql = jest.spyOn(SQLite, 'openDatabase');
+  failNextSql(/^PRAGMA user_version/);
+  await expect(openRuntimeItemStore()).rejects.toThrow('injected persistence failure');
+  const identity = jest.fn(() => 'unit-owner-intent');
+  const laterIdentity = jest.fn(() => 'unit-owner-next-intent');
+  const first = openRuntimeItemStore(identity);
+  const second = openRuntimeItemStore(laterIdentity);
+  const [store, headlessStore] = await Promise.all([first, second]);
+  expect(openingSql).toHaveBeenCalledTimes(2); // One failed init, one shared retry.
+  const seed = {id: 'unit-owner', title: 'Saved owner', completed: false, version: 1, updatedAt: 1700000000000};
+  await store.replaceSnapshot([seed], 1700000000000);
+  let committed!: () => void;
+  const didCommit = new Promise<void>(resolve => {committed = resolve;});
+  let release!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  const original = store.mutate.bind(store);
+  const delayed = jest.spyOn(store, 'mutate').mockImplementation(async (...args) => {
+    const rows = await original(...args); committed(); await held; return rows;
+  });
+  const oldSave = store.mutate({type: 'rename', id: seed.id, title: 'Owner edit', now: 1700000600000});
+  try {
+    await didCommit;
+    const operation = await headlessStore.prepareNext();
+    expect(operation?.clientMutationId).toBe('unit-owner-intent');
+    const canonical = {...seed, title: 'Owner edit', version: 2, updatedAt: 1700000100000};
+    await headlessStore.acknowledge(operation!, {item: canonical});
+    expect(await headlessStore.read()).toEqual([canonical]);
+    expect(await headlessStore.readPending()).toEqual([]);
+    expect(await headlessStore.readLastSuccessfulRefresh()).toBe(1700000000000);
+    expect(identity).toHaveBeenCalledTimes(1);
+    expect(laterIdentity).not.toHaveBeenCalled();
+    await headlessStore.mutate({type: 'rename', id: seed.id, title: 'Next caller edit', now: 1700000601000});
+    expect((await headlessStore.readPending())[0]).toMatchObject({clientMutationId: 'unit-owner-next-intent',
+      payload: {title: 'Next caller edit', baseVersion: 2}});
+    expect(identity).toHaveBeenCalledTimes(1);
+    expect(laterIdentity).toHaveBeenCalledTimes(1);
+  } finally {release(); await oldSave; delayed.mockRestore(); openingSql.mockRestore();}
+});
diff --git a/android/app/build.gradle b/android/app/build.gradle
index b320563..a430f69 100644
--- a/android/app/build.gradle
+++ b/android/app/build.gradle
@@ -25,6 +25,9 @@ android {
 dependencies {
     implementation("com.facebook.react:react-android:0.76.9")
     implementation("com.facebook.react:hermes-android:0.76.9")
+    implementation("androidx.work:work-runtime:2.9.1")
+    implementation("androidx.concurrent:concurrent-futures:1.1.0")
+    testImplementation("junit:junit:4.13.2")
     androidTestImplementation("androidx.test:runner:1.6.2")
     androidTestImplementation("androidx.test:core:1.6.1")
     androidTestImplementation("androidx.test.ext:junit:1.2.1")
diff --git a/android/app/src/main/AndroidManifest.xml b/android/app/src/main/AndroidManifest.xml
index 002b0cf..f09a3fd 100644
--- a/android/app/src/main/AndroidManifest.xml
+++ b/android/app/src/main/AndroidManifest.xml
@@ -1,6 +1,9 @@
-<manifest xmlns:android="http://schemas.android.com/apk/res/android">
+<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">
     <uses-permission android:name="android.permission.INTERNET" />
     <application android:name=".MainApplication" android:label="Offline Item Tracker" android:allowBackup="false" android:theme="@style/AppTheme" android:supportsRtl="true" android:networkSecurityConfig="@xml/network_security_config">
+        <provider android:name="androidx.startup.InitializationProvider" android:authorities="${applicationId}.androidx-startup" android:exported="false" tools:node="merge">
+            <meta-data android:name="androidx.work.WorkManagerInitializer" tools:node="remove" />
+        </provider>
         <activity android:name=".MainActivity" android:exported="true" android:windowSoftInputMode="adjustResize">
             <intent-filter>
                 <action android:name="android.intent.action.MAIN" />
diff --git a/android/app/src/main/java/com/mse/reactnative/BackgroundCycle.kt b/android/app/src/main/java/com/mse/reactnative/BackgroundCycle.kt
new file mode 100644
index 0000000..deb987c
--- /dev/null
+++ b/android/app/src/main/java/com/mse/reactnative/BackgroundCycle.kt
@@ -0,0 +1,20 @@
+package com.mse.reactnative
+
+// The allowance is persisted before transport. A killed reservation may consume
+// a slot without a request; it must never grant more than three automatic sends.
+data class BackgroundCycle(val id: String, val attempts: Int = 0, val status: String = "active") {
+    init {
+        require(id.isNotEmpty() && attempts in 0..3)
+        require(status in setOf("active", "deferred", "complete"))
+    }
+
+    fun reserve(): BackgroundCycle? =
+        if (status == "active" && attempts < 3) copy(attempts = attempts + 1) else null
+
+    fun finish(outcome: String): BackgroundCycle = when (outcome) {
+        "success" -> copy(status = "complete")
+        "retry" -> copy(status = if (attempts < 3) "active" else "deferred")
+        "failure" -> copy(status = "deferred")
+        else -> throw IllegalArgumentException("Unknown background result")
+    }
+}
diff --git a/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt b/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt
new file mode 100644
index 0000000..517d2f6
--- /dev/null
+++ b/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt
@@ -0,0 +1,303 @@
+package com.mse.reactnative
+
+import android.content.Context
+import android.os.Handler
+import android.os.Looper
+import android.os.Process
+import android.util.Log
+import androidx.concurrent.futures.CallbackToFutureAdapter
+import androidx.work.BackoffPolicy
+import androidx.work.Constraints
+import androidx.work.Data
+import androidx.work.ExistingWorkPolicy
+import androidx.work.ListenableWorker
+import androidx.work.NetworkType
+import androidx.work.OneTimeWorkRequest
+import androidx.work.WorkManager
+import androidx.work.WorkerParameters
+import com.facebook.react.ReactApplication
+import com.facebook.react.ReactInstanceManager
+import com.facebook.react.ReactPackage
+import com.facebook.react.bridge.Arguments
+import com.facebook.react.bridge.NativeModule
+import com.facebook.react.bridge.Promise
+import com.facebook.react.bridge.ReactApplicationContext
+import com.facebook.react.bridge.ReactContext
+import com.facebook.react.bridge.ReactContextBaseJavaModule
+import com.facebook.react.bridge.ReactMethod
+import com.facebook.react.bridge.WritableMap
+import com.facebook.react.jstasks.HeadlessJsTaskConfig
+import com.facebook.react.jstasks.HeadlessJsTaskContext
+import com.facebook.react.jstasks.HeadlessJsTaskEventListener
+import com.facebook.react.modules.core.DeviceEventManagerModule
+import com.facebook.react.uimanager.ViewManager
+import com.google.common.util.concurrent.ListenableFuture
+import org.json.JSONObject
+import java.util.UUID
+import java.util.concurrent.TimeUnit
+
+internal object BackgroundSync {
+    const val UNIQUE_WORK = "item-uploads"
+    private const val PREFS = "item-background-sync"
+    private val lock = Any()
+    private val runs = mutableMapOf<String, BackgroundWorker>()
+
+    fun read(context: Context): BackgroundCycle? {
+        val prefs = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
+        val id = prefs.getString("cycleId", null) ?: return null
+        return BackgroundCycle(id, prefs.getInt("attempts", 0), prefs.getString("status", "active")!!)
+    }
+
+    private fun save(context: Context, cycle: BackgroundCycle) {
+        check(context.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit()
+            .putString("cycleId", cycle.id).putInt("attempts", cycle.attempts).putString("status", cycle.status).commit()) {
+            "Background allowance was not durably committed"
+        }
+    }
+
+    fun state(context: Context): WritableMap = synchronized(lock) {
+        val cycle = read(context)
+        Arguments.createMap().apply {
+            putString("cycleId", cycle?.id)
+            putInt("attempts", cycle?.attempts ?: 0)
+            putString("status", cycle?.status ?: "none")
+        }
+    }
+
+    private fun emit(context: Context, name: String, value: WritableMap) {
+        val react = (context.applicationContext as ReactApplication).reactNativeHost.reactInstanceManager.currentReactContext
+        if (react?.hasActiveReactInstance() == true) {
+            react.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java).emit(name, value)
+        }
+    }
+
+    fun changed(context: Context) = emit(context, "BackgroundSyncChanged", state(context))
+
+    fun schedule(context: Context): WritableMap {
+        val operation = synchronized(lock) {
+            val previous = read(context)
+            if (previous?.status == "deferred" || (previous?.status == "active" && previous.attempts >= 3)) {
+                if (previous.status != "deferred") save(context, previous.copy(status = "deferred"))
+                return state(context)
+            }
+            val fresh = previous == null || previous.status == "complete"
+            val cycle = if (fresh) BackgroundCycle(UUID.randomUUID().toString()) else previous!!
+            save(context, cycle)
+            val request = OneTimeWorkRequest.Builder(BackgroundWorker::class.java)
+                .setInputData(Data.Builder().putString("cycleId", cycle.id).build())
+                .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
+                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS).build()
+            // Repeated active requests keep one chain. If a completed cycle's
+            // final Worker is still returning, append the new cycle safely.
+            WorkManager.getInstance(context).enqueueUniqueWork(UNIQUE_WORK,
+                if (fresh) ExistingWorkPolicy.APPEND_OR_REPLACE else ExistingWorkPolicy.KEEP, request)
+        }
+        // Registration completion only: never wait for worker execution in JS.
+        operation.result.get()
+        changed(context)
+        return state(context)
+    }
+
+    fun prepareManual(context: Context): WritableMap {
+        synchronized(lock) {
+            runs.values.toList().forEach { it.cancel(false, true) }
+            read(context)?.let { save(context, it.copy(status = "complete")) }
+        }
+        WorkManager.getInstance(context).cancelUniqueWork(UNIQUE_WORK).result.get()
+        changed(context)
+        return state(context)
+    }
+
+    fun register(worker: BackgroundWorker): Boolean = synchronized(lock) {
+        val cycle = read(worker.applicationContext)
+        if (cycle?.id != worker.cycleId) return false
+        runs[worker.token] = worker
+        true
+    }
+
+    fun active(token: String): Boolean = synchronized(lock) {
+        val worker = runs[token] ?: return false
+        !worker.cancelled && read(worker.applicationContext)?.id == worker.cycleId
+    }
+
+    fun reserve(token: String): Boolean = synchronized(lock) {
+        val worker = runs[token] ?: return false
+        if (!active(token) || worker.httpInFlight) return false
+        val next = read(worker.applicationContext)?.reserve() ?: return false
+        save(worker.applicationContext, next)
+        worker.httpInFlight = true
+        worker.trace("HTTP_RESERVED", next.attempts)
+        true
+    }
+
+    fun requestFinished(token: String) = synchronized(lock) {
+        runs[token]?.let { worker ->
+            worker.httpInFlight = false
+            worker.trace("HTTP_FINISHED", read(worker.applicationContext)?.attempts ?: 0)
+            if (worker.cancelled) worker.finishCancellation()
+        }
+    }
+
+    fun complete(token: String, outcome: String): Boolean = synchronized(lock) {
+        val worker = runs[token] ?: return false
+        if (!active(token) || worker.httpInFlight) return false
+        val cycle = read(worker.applicationContext)!!
+        val finished = cycle.finish(outcome)
+        save(worker.applicationContext, finished)
+        worker.trace("RESULT_$outcome", finished.attempts)
+        worker.finish(if (finished.status == "complete") ListenableWorker.Result.success()
+            else if (finished.status == "active") ListenableWorker.Result.retry() else ListenableWorker.Result.failure())
+        changed(worker.applicationContext)
+        true
+    }
+
+    fun abort(worker: BackgroundWorker) = synchronized(lock) {
+        if (runs[worker.token] !== worker) return
+        worker.cancelled = true
+        emit(worker.applicationContext, "BackgroundSyncCancelled", Arguments.createMap().apply { putString("token", worker.token) })
+        if (!worker.httpInFlight) worker.finishCancellation()
+    }
+
+    fun remove(worker: BackgroundWorker) = synchronized(lock) {
+        if (runs[worker.token] === worker) runs.remove(worker.token)
+    }
+
+    fun cancelledResult(worker: BackgroundWorker, retry: Boolean): ListenableWorker.Result = synchronized(lock) {
+        val current = read(worker.applicationContext)
+        if (current?.id != worker.cycleId) return ListenableWorker.Result.failure()
+        val next = current.finish(if (retry) "retry" else "failure")
+        save(worker.applicationContext, next)
+        changed(worker.applicationContext)
+        if (next.status == "active") ListenableWorker.Result.retry() else ListenableWorker.Result.failure()
+    }
+}
+
+class BackgroundWorker(context: Context, parameters: WorkerParameters) : ListenableWorker(context, parameters), HeadlessJsTaskEventListener {
+    val token: String = UUID.randomUUID().toString()
+    val cycleId: String = inputData.getString("cycleId") ?: ""
+    @Volatile var httpInFlight = false
+    @Volatile var cancelled = false
+    @Volatile private var retryCancellation = false
+    @Volatile private var platformStopped = false
+    private var completer: CallbackToFutureAdapter.Completer<Result>? = null
+    private var manager: ReactInstanceManager? = null
+    private var listener: ReactInstanceManager.ReactInstanceEventListener? = null
+    private var tasks: HeadlessJsTaskContext? = null
+    private var taskId = -1
+    private var jsStarted = false
+    private val main = Handler(Looper.getMainLooper())
+    private val timeout = Runnable { cancel(httpInFlight, false) }
+
+    fun trace(phase: String, attempts: Int = 0) {
+        // Diagnostics must not strand a durably reserved slot before JS gets
+        // its authorization, even if a debug clock file becomes unreadable.
+        try {
+            Log.i("MSEBackground", JSONObject().put("phase", phase).put("pid", Process.myPid())
+                .put("workId", id.toString()).put("cycleId", cycleId).put("token", token)
+                .put("workerAttempt", runAttemptCount).put("reservedAttempts", attempts)
+                .put("clock", WorkManager.getInstance(applicationContext).configuration.clock.currentTimeMillis()).toString())
+        } catch (error: Exception) { Log.e("MSEBackground", "Worker trace unavailable", error) }
+    }
+
+    override fun startWork(): ListenableFuture<Result> = CallbackToFutureAdapter.getFuture { completion ->
+        completer = completion
+        if (!BackgroundSync.register(this)) completion.set(Result.failure())
+        else try {
+            trace("START", BackgroundSync.read(applicationContext)?.attempts ?: 0)
+            main.postDelayed(timeout, 60000)
+            val host = (applicationContext as ReactApplication).reactNativeHost
+            manager = host.reactInstanceManager
+            val existing = manager!!.currentReactContext
+            if (existing != null) startJs(existing)
+            else {
+                listener = ReactInstanceManager.ReactInstanceEventListener { context ->
+                    listener?.let { manager?.removeReactInstanceEventListener(it) }
+                    listener = null
+                    startJs(context)
+                }
+                manager!!.addReactInstanceEventListener(listener!!)
+                // Context publication can race listener registration. Both
+                // paths are on the UI thread; startJs guards duplicate delivery.
+                val ready = manager!!.currentReactContext
+                if (ready != null) {
+                    listener?.let { manager?.removeReactInstanceEventListener(it) }
+                    listener = null
+                    startJs(ready)
+                } else if (!manager!!.hasStartedCreatingInitialContext()) manager!!.createReactContextInBackground()
+            }
+        } catch (error: Exception) {
+            Log.e("MSEBackground", "Headless context initialization failed", error)
+            cancel(false, false)
+        }
+        "Item background sync $id"
+    }
+
+    private fun startJs(context: ReactContext) {
+        if (jsStarted || !BackgroundSync.active(token)) return
+        jsStarted = true
+        try {
+            tasks = HeadlessJsTaskContext.getInstance(context)
+            tasks!!.addTaskEventListener(this)
+            val input = Arguments.createMap().apply {
+                putString("token", token); putString("cycleId", cycleId); putString("workId", id.toString())
+            }
+            // This Worker owns timeout/outcome. RN's timeout/finish has no result,
+            // and default RN in-process retry must not layer over WorkManager.
+            taskId = tasks!!.startTask(HeadlessJsTaskConfig("ItemBackgroundSync", input, 0, true))
+        } catch (error: Exception) {
+            Log.e("MSEBackground", "Headless task initialization failed", error)
+            cancel(false, false)
+        }
+    }
+
+    override fun onHeadlessJsTaskStart(taskId: Int) = Unit
+    override fun onHeadlessJsTaskFinish(taskId: Int) {
+        if (this.taskId == taskId && completer != null) cancel(false, false)
+    }
+
+    fun cancel(retry: Boolean, stopped: Boolean) {
+        if (!cancelled) retryCancellation = retry
+        if (stopped) platformStopped = true
+        BackgroundSync.abort(this)
+    }
+
+    fun finishCancellation() {
+        // A timeout cannot return retry until JS confirms its fetch settled
+        // after abort. A platform stop already cancelled the future; JS's shared
+        // lock keeps any replacement invocation behind that aborted request.
+        if (platformStopped) finish(null)
+        else finish(BackgroundSync.cancelledResult(this, retryCancellation))
+    }
+
+    fun finish(result: Result?) {
+        val completion = completer
+        completer = null
+        BackgroundSync.remove(this)
+        main.removeCallbacks(timeout)
+        listener?.let { manager?.removeReactInstanceEventListener(it) }
+        listener = null
+        tasks?.removeTaskEventListener(this)
+        if (result != null) completion?.set(result)
+    }
+
+    override fun onStopped() { cancel(false, true) }
+}
+
+class BackgroundSyncModule(context: ReactApplicationContext) : ReactContextBaseJavaModule(context) {
+    override fun getName() = "BackgroundSync"
+    private fun answer(promise: Promise, action: () -> Any?) {
+        try { promise.resolve(action()) } catch (error: Exception) { promise.reject("BACKGROUND_SYNC", error) }
+    }
+    @ReactMethod fun getState(promise: Promise) = answer(promise) { BackgroundSync.state(reactApplicationContext) }
+    @ReactMethod fun schedule(promise: Promise) = answer(promise) { BackgroundSync.schedule(reactApplicationContext) }
+    @ReactMethod fun prepareManual(promise: Promise) = answer(promise) { BackgroundSync.prepareManual(reactApplicationContext) }
+    @ReactMethod fun isActive(token: String, promise: Promise) = answer(promise) { BackgroundSync.active(token) }
+    @ReactMethod fun reserve(token: String, promise: Promise) = answer(promise) { BackgroundSync.reserve(token) }
+    @ReactMethod fun requestFinished(token: String, promise: Promise) = answer(promise) { BackgroundSync.requestFinished(token); null }
+    @ReactMethod fun complete(token: String, outcome: String, promise: Promise) = answer(promise) { BackgroundSync.complete(token, outcome) }
+}
+
+class BackgroundSyncPackage : ReactPackage {
+    override fun createNativeModules(context: ReactApplicationContext): List<NativeModule> = listOf(BackgroundSyncModule(context))
+    override fun createViewManagers(context: ReactApplicationContext): List<ViewManager<*, *>> = emptyList()
+}
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index a14bff0..400109a 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -25,6 +25,9 @@ class MainActivity : ReactActivity() {
                     if (intent.getBooleanExtra("m09FixedIdentity", false)) {
                         putString("testIdentityPrefix", "death")
                     }
+                    if (intent.getBooleanExtra("m10FixedIdentity", false)) {
+                        putString("testIdentityPrefix", "work")
+                    }
                     intent.getStringExtra("m07MutationIdentity")?.let {
                         putString("testMutationIdentity", it)
                     }
diff --git a/android/app/src/main/java/com/mse/reactnative/MainApplication.kt b/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
index 9282466..75fe170 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
@@ -1,6 +1,10 @@
 package com.mse.reactnative
 
 import android.app.Application
+import android.util.Log
+import androidx.work.Clock
+import androidx.work.Configuration
+import androidx.work.RunnableScheduler
 import com.facebook.react.PackageList
 import com.facebook.react.ReactApplication
 import com.facebook.react.ReactHost
@@ -9,11 +13,13 @@ import com.facebook.react.ReactPackage
 import com.facebook.react.defaults.DefaultReactHost.getDefaultReactHost
 import com.facebook.react.defaults.DefaultReactNativeHost
 import com.facebook.react.soloader.OpenSourceMergedSoMapping
+import com.facebook.react.modules.network.OkHttpClientProvider
 import com.facebook.soloader.SoLoader
+import java.io.File
 
-class MainApplication : Application(), ReactApplication {
+class MainApplication : Application(), ReactApplication, Configuration.Provider {
     override val reactNativeHost: ReactNativeHost = object : DefaultReactNativeHost(this) {
-        override fun getPackages(): List<ReactPackage> = PackageList(this).packages
+        override fun getPackages(): List<ReactPackage> = PackageList(this).packages.apply { add(BackgroundSyncPackage()) }
         override fun getJSMainModuleName(): String = "index"
         override fun getUseDeveloperSupport(): Boolean = false
         override val isNewArchEnabled: Boolean = false
@@ -23,8 +29,32 @@ class MainApplication : Application(), ReactApplication {
     override val reactHost: ReactHost
         get() = getDefaultReactHost(applicationContext, reactNativeHost)
 
+    override val workManagerConfiguration: Configuration
+        get() {
+            val builder = Configuration.Builder().setMinimumLoggingLevel(if (BuildConfig.DEBUG) Log.DEBUG else Log.INFO)
+            val testClock = File(filesDir, "m10-work-clock")
+            if (BuildConfig.DEBUG && testClock.isFile) {
+                // Test-only durable clock: real WorkDatabase/SystemJobScheduler
+                // remain installed. Host control atomically replaces this file.
+                builder.setClock(Clock { testClock.readText().trim().toLong() })
+                builder.setRunnableScheduler(object : RunnableScheduler {
+                    override fun scheduleWithDelay(delayInMillis: Long, runnable: Runnable) {
+                        Log.d("MSEBackground", "TEST_DELAY_HELD milliseconds=$delayInMillis")
+                    }
+                    override fun cancel(runnable: Runnable) = Unit
+                })
+            }
+            return builder.build()
+        }
+
     override fun onCreate() {
         super.onCreate()
         SoLoader.init(this, OpenSourceMergedSoMapping)
+        // Transport must not spend an invisible extra HTTP attempt beneath the
+        // durable allowance. Foreground still uses the same fetch/ACK protocol.
+        OkHttpClientProvider.setOkHttpClientFactory {
+            OkHttpClientProvider.createClientBuilder().retryOnConnectionFailure(false)
+                .followRedirects(false).followSslRedirects(false).build()
+        }
     }
 }
diff --git a/android/app/src/test/java/com/mse/reactnative/BackgroundCycleTest.kt b/android/app/src/test/java/com/mse/reactnative/BackgroundCycleTest.kt
new file mode 100644
index 0000000..2b8559e
--- /dev/null
+++ b/android/app/src/test/java/com/mse/reactnative/BackgroundCycleTest.kt
@@ -0,0 +1,27 @@
+package com.mse.reactnative
+
+import org.junit.Assert.*
+import org.junit.Test
+
+class BackgroundCycleTest {
+    @Test fun allowanceRestorationDoesNotResetExhaustion() {
+        var cycle = BackgroundCycle("durable-cycle")
+        repeat(3) { expected ->
+            cycle = cycle.reserve()!!
+            assertEquals(expected + 1, cycle.attempts)
+            // Same fields read after a process loss must retain the allowance.
+            cycle = BackgroundCycle(cycle.id, cycle.attempts, cycle.status)
+        }
+        assertNull(cycle.reserve())
+        assertEquals("deferred", cycle.finish("retry").status)
+        assertNull(cycle.finish("retry").reserve())
+    }
+
+    @Test fun committedSuccessAndTerminalFailureAreNotRetry() {
+        val last = BackgroundCycle("cycle", 3)
+        assertEquals("complete", last.finish("success").status)
+        assertNull(last.finish("success").reserve())
+        assertEquals("active", BackgroundCycle("cycle", 1).finish("retry").status)
+        assertEquals("deferred", BackgroundCycle("cycle", 1).finish("failure").status)
+    }
+}
diff --git a/index.js b/index.js
index c3af1ff..2da1552 100644
--- a/index.js
+++ b/index.js
@@ -1,5 +1,7 @@
 import {AppRegistry} from 'react-native';
 import App from './src/App';
+import {runBackgroundTask} from './src/backgroundSync';
 import {name} from './app.json';
 
 AppRegistry.registerComponent(name, () => App);
+AppRegistry.registerHeadlessTask('ItemBackgroundSync', () => runBackgroundTask);
diff --git a/scripts/verify_m10.py b/scripts/verify_m10.py
new file mode 100644
index 0000000..7dd22b6
--- /dev/null
+++ b/scripts/verify_m10.py
@@ -0,0 +1,615 @@
+#!/usr/bin/env python3
+"""One root-authorized M10 case, real persistent WorkManager/OS dispatch.
+
+No TestInitHelper, direct Worker, forced job run, or app entry across process
+loss. The only post-loss write is the declared DEBUG clock control file.
+Case A and B are separate invocations so root charges each before execution.
+"""
+import argparse
+import base64
+import datetime
+import hashlib
+import io
+import json
+import os
+from pathlib import Path
+import re
+import shutil
+import socket
+import sqlite3
+import subprocess
+import tarfile
+import time
+from urllib.error import HTTPError
+from urllib.request import Request, urlopen
+import xml.etree.ElementTree as ET
+from verify_m07 import package_in_live_activities
+from verify_m10_baseline import PACKAGE, URL, NETWORK_KEYS, ONLINE, OFFLINE
+
+WORKER = PACKAGE + ".BackgroundWorker"
+SERVICE = "androidx.work.impl.background.systemjob.SystemJobService"
+UNIQUE = "item-uploads"
+CLOCK = "files/m10-work-clock"
+WORK_DB = "no_backup/androidx.work.workdb"
+PREFS = "shared_prefs/item-background-sync.xml"
+
+
+class SnapshotAcquisitionError(Exception):
+    """A live file copy was incomplete, not a logical state assertion failure."""
+
+
+def sha(path):
+    return hashlib.sha256(Path(path).read_bytes()).hexdigest()
+
+
+def registered_jobs(dump):
+    # API34 JobStatus.printUniqueId: #uid/id for null namespace, ns:uid/id otherwise.
+    jobs = []
+    for line in dump.splitlines():
+        found = re.match(r"\s*JOB (?:#|([^:\s]+):)([^/\s]+)/(\d+): (.*)$", line)
+        if found and PACKAGE + "/" + SERVICE in found[4]:
+            jobs.append({"namespace": found[1], "uid": found[2], "id": int(found[3]), "header": line.strip()})
+    return jobs
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--case", required=True, choices=("A", "B"))
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--node", default="node")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    args = parser.parse_args()
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs = json.loads((root / "verification/M10-inputs.json").read_text())
+    commands, controls = [], []
+    result = {"status": "RUNNING", "case": args.case, "hostPid": os.getpid(), "serial": args.serial,
+              "apkSha256": sha(args.apk), "inputsSha256": sha(root / "verification/M10-inputs.json"),
+              "harnessSha256": sha(__file__), "fixtureSha256": sha(root / "fixture/server.cjs"),
+              "caseBudget": "Root charges this exact case before invocation; no retries or warmups"}
+    fixture, original_network = None, None
+    started = time.monotonic()
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(label, *parts, check=True, binary=False, timeout=60):
+        command = [args.adb, "-s", args.serial, *parts]
+        entry = {"label": label, "command": command, "timeoutSeconds": timeout, "startedAt": int(time.time() * 1000)}
+        before = time.monotonic()
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
+                name = f"adb-{len(commands):04d}-{label}.{stream}"
+                (evidence / name).write_bytes(data)
+                entry[stream] = name
+        entry.update(elapsedSeconds=time.monotonic() - before, endedAt=int(time.time() * 1000))
+        commands.append(entry)
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        assert entry["exit"] is not None, entry
+        if check:
+            assert entry["exit"] == 0, entry
+        return raw if binary else raw.decode(errors="replace").strip()
+
+    def remote(path="/__m10-state", body=None):
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
+        deadline = time.monotonic() + inputs["networkTimeoutSeconds"]
+        while True:
+            settings = {key: adb(label + "-" + key, "shell", "settings", "get", "global", key) for key in NETWORK_KEYS}
+            connectivity = adb(label + "-connectivity", "shell", "dumpsys", "connectivity")
+            offline = "Active default network: none" in connectivity
+            if expected is None or (settings == expected and offline == (expected == OFFLINE)):
+                assert offline == (settings == OFFLINE)
+                return settings
+            assert time.monotonic() < deadline, (expected, settings)
+            time.sleep(0.1)
+
+    def quiet_reset():
+        start, quiet = time.monotonic(), None
+        observations = result["resetObservations"] = []
+        while True:
+            left = inputs["resetTimeoutSeconds"] - (time.monotonic() - start)
+            assert left > 0, "Activity teardown did not settle within 10s"
+            pid = adb("setup-pid", "shell", "pidof", PACKAGE, check=False, timeout=left)
+            assert commands[-1]["exit"] in (0, 1)
+            left = inputs["resetTimeoutSeconds"] - (time.monotonic() - start)
+            assert left > 0
+            activities = adb("setup-activities", "shell", "dumpsys", "activity", "activities", timeout=left)
+            present = package_in_live_activities(activities)
+            now = time.monotonic()
+            observations.append({"elapsedSeconds": now - start, "pid": pid, "liveActivity": present})
+            save()
+            assert now - start < inputs["resetTimeoutSeconds"]
+            if not pid and not present:
+                quiet = now if quiet is None else quiet
+                if now - quiet >= inputs["resetQuietSeconds"]:
+                    return
+            else:
+                quiet = None
+            time.sleep(0.1)
+
+    def snapshot(name=None):
+        adb("ui-dump", "shell", "uiautomator", "dump", "/sdcard/mse-m10-ui.xml")
+        xml = adb("ui-read", "exec-out", "cat", "/sdcard/mse-m10-ui.xml")
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
+    def launch(label, *extras):
+        output = adb(label, "shell", "am", "start", "-W", "-n", PACKAGE + "/.MainActivity", *extras)
+        assert "Status: ok" in output, output
+        pid = adb(label + "-pid", "shell", "pidof", PACKAGE)
+        assert re.fullmatch(r"\d+", pid), pid
+        return pid
+
+    def resumed_activity(label):
+        deadline = time.monotonic() + 10
+        while True:
+            dump = adb(label, "shell", "dumpsys", "activity", "activities")
+            current = set(re.findall(r"(?:mResumedActivity:|topResumedActivity=)\s*ActivityRecord\{([^ ]+) [^}\n]*"
+                                     + re.escape(PACKAGE) + r"/\.MainActivity\b[^}\n]*\bt(\d+)\b", dump))
+            if len(current) == 1:
+                record, task = current.pop()
+                return {"record": record, "task": task, "commandIndex": len(commands) - 1}
+            assert time.monotonic() < deadline, current
+            time.sleep(0.1)
+
+    def no_process(label):
+        deadline = time.monotonic() + 5
+        while True:
+            pid = adb(label, "shell", "pidof", PACKAGE, check=False)
+            assert commands[-1]["exit"] in (0, 1)
+            if not pid:
+                return {"commandIndex": len(commands) - 1, "at": int(time.time() * 1000), "pid": ""}
+            assert time.monotonic() < deadline, pid
+            time.sleep(0.1)
+
+    def not_stopped(label):
+        dump = adb(label, "shell", "dumpsys", "package", PACKAGE)
+        state = re.search(r"(?m)^\s*User 0:[^\n]*", dump)
+        assert state and re.search(r"\bstopped=false\b", state.group()), dump
+        return state.group().strip()
+
+    def database_copy(label, device_path, deadline):
+        def remaining():
+            value = deadline - time.monotonic()
+            if value <= 0:
+                raise SnapshotAcquisitionError("Native copy observation deadline elapsed")
+            return value
+        parent, basename = device_path.rsplit("/", 1)
+        files = adb(label + "-files", "shell", "run-as", PACKAGE, "ls", parent, timeout=remaining()).splitlines()
+        names = [device_path + suffix for suffix in ("", "-wal", "-shm") if basename + suffix in files]
+        assert device_path in names
+        raw = adb(label + "-native", "exec-out", "run-as", PACKAGE, "tar", "-cf", "-", *names,
+                  binary=True, check=False, timeout=remaining())
+        archive = evidence / (label + ".tar")
+        archive.write_bytes(raw)
+        if commands[-1]["exit"] != 0:
+            raise SnapshotAcquisitionError("Native tar failed; raw archive/stdout/stderr retained")
+        native, analysis = evidence / (label + "-native"), evidence / (label + "-analysis")
+        native.mkdir(); analysis.mkdir()
+        hashes = {}
+        with tarfile.open(fileobj=io.BytesIO(raw)) as saved:
+            for member in saved:
+                assert member.isfile() and member.name in names
+                name = basename + member.name.removeprefix(device_path)
+                (native / name).write_bytes(saved.extractfile(member).read())
+                hashes[name] = sha(native / name)
+                shutil.copy2(native / name, analysis / name)
+        with sqlite3.connect(f"file:{analysis / basename}?mode=ro", uri=True) as db:
+            db.row_factory = sqlite3.Row
+            integrity = [row[0] for row in db.execute("PRAGMA integrity_check")]
+            if integrity != ["ok"]:
+                raise SnapshotAcquisitionError("Copied SQLite integrity: " + repr(integrity))
+            schema = [dict(row) for row in db.execute("SELECT name,sql FROM sqlite_master WHERE type='table' ORDER BY name")]
+            tables = {}
+            for table in schema:
+                assert re.fullmatch(r"[A-Za-z_][A-Za-z_0-9]*", table["name"])
+                tables[table["name"]] = [dict(row) for row in db.execute('SELECT * FROM "' + table["name"] + '" ORDER BY rowid')]
+            value = {"schemaVersion": db.execute("PRAGMA user_version").fetchone()[0], "schema": schema, "tables": tables}
+        encoded = json.loads(json.dumps(value, default=lambda blob: {"base64": base64.b64encode(blob).decode()}))
+        (evidence / (label + ".json")).write_text(json.dumps(encoded, indent=2) + "\n")
+        assert all(sha(native / name) == digest for name, digest in hashes.items())
+        result.setdefault("nativeDatabases", []).append({"name": label, "devicePath": device_path,
+            "archiveSha256": sha(archive), "nativeFileSha256": hashes})
+        save()
+        return value["tables"]
+
+    def database(label, device_path, deadline=None):
+        # A WAL/checkpoint may race a read-only tar copy. Preserve every failed
+        # acquisition and retry ONLY acquisition errors within this observation.
+        # A valid copied state that violates an expectation is never retried here.
+        deadline = deadline if deadline is not None else time.monotonic() + 5
+        attempt = 0
+        while True:
+            attempt += 1
+            name = label if attempt == 1 else f"{label}-acquisition-{attempt}"
+            try:
+                return database_copy(name, device_path, deadline)
+            except (SnapshotAcquisitionError, tarfile.TarError, sqlite3.DatabaseError) as error:
+                result.setdefault("nativeAcquisitionFailures", []).append({"name": name, "devicePath": device_path,
+                    "error": repr(error), "lastCommandIndex": len(commands) - 1, "at": int(time.time() * 1000)})
+                save()
+                if time.monotonic() >= deadline:
+                    raise
+                time.sleep(0.1)
+
+    def preferences(label):
+        raw = adb(label, "exec-out", "run-as", PACKAGE, "cat", PREFS)
+        (evidence / (label + ".xml")).write_text(raw + "\n")
+        value = {node.get("name"): int(node.get("value")) if node.tag == "int" else node.text for node in ET.fromstring(raw)}
+        assert set(value) == {"cycleId", "attempts", "status"}, value
+        return value
+
+    def clock(label, value):
+        path = evidence / (label + ".clock")
+        path.write_text(str(value) + "\n")
+        temporary = "/data/local/tmp/mse-m10-clock-" + str(os.getpid())
+        adb(label + "-push", "push", str(path), temporary)
+        adb(label + "-copy", "shell", "run-as", PACKAGE, "cp", temporary, CLOCK + ".next")
+        adb(label + "-replace", "shell", "run-as", PACKAGE, "mv", CLOCK + ".next", CLOCK)
+        adb(label + "-remove-temporary", "shell", "rm", temporary)
+        assert adb(label + "-read", "exec-out", "run-as", PACKAGE, "cat", CLOCK) == str(value)
+        result.setdefault("clockChanges", []).append({"value": value, "lastCommandIndex": len(commands) - 1})
+        save()
+
+    def work(tables):
+        specs = tables["WorkSpec"]
+        assert len(specs) == 1 and specs[0]["worker_class_name"] == WORKER, specs
+        spec = specs[0]
+        assert spec["required_network_type"] == 1 and spec["backoff_policy"] == 0 and spec["backoff_delay_duration"] == 10000
+        assert tables["WorkName"] == [{"name": UNIQUE, "work_spec_id": spec["id"]}]
+        if "workId" in result:
+            assert spec["id"] == result["workId"] and spec["input"] == bytes.fromhex(result["workInputHex"])
+        return spec
+
+    def jobs(label):
+        dump = adb(label, "shell", "dumpsys", "jobscheduler", PACKAGE)
+        return registered_jobs(dump)
+
+    def registered(label, tables):
+        spec = work(tables)
+        ids = tables["SystemIdInfo"]
+        assert len(ids) == 1 and ids[0]["work_spec_id"] == spec["id"], ids
+        assert ids[0]["generation"] == spec["generation"] and spec["schedule_requested_at"] != -1
+        actual = jobs(label)
+        assert len(actual) == 1 and actual[0]["id"] == ids[0]["system_id"], (actual, ids)
+        assert actual[0]["namespace"] is None, actual
+        return actual[0]
+
+    def run_job(label, job, unmet=False):
+        # Omit -n for actual null namespace. Never use -f or a direct Worker API.
+        assert job["namespace"] is None
+        output = adb(label, "shell", "cmd", "jobscheduler", "run", "-u", "0", PACKAGE, str(job["id"]), check=False)
+        entry = commands[-1]
+        stderr = (evidence / entry["stderr"]).read_text() if entry["stderr"] else ""
+        if unmet:
+            assert entry["exit"] != 0 and "constraints" in (output + stderr).lower(), (entry, output, stderr)
+        else:
+            # Natural OS dispatch can race this nonforced test command. An old
+            # terminal job may already be gone; actual SQL/HTTP observations win.
+            assert entry["exit"] == 0 or "could not find job" in (output + stderr).lower(), (entry, output, stderr)
+        return {"commandIndex": len(commands) - 1, "exit": entry["exit"], "job": job}
+
+    def no_http():
+        state = remote()
+        assert state == {"items": [], "nextTimestamp": inputs["nextTimestamp"], "requests": [], "case": args.case, "httpAttempts": 0}, state
+        assert remote("/__mutation-state") == {"appliedCount": 0, "duplicateCount": 0, "conflictCount": 0, "hashRejectedCount": 0, "attempts": []}
+        return state
+
+    def wait_attempt(number, expected_clock):
+        deadline = time.monotonic() + 45
+        ordinal = 0
+        while True:
+            ordinal += 1
+            tables = database(f"attempt-{number}-observe-{ordinal}", WORK_DB, deadline)
+            spec = work(tables)
+            state = remote()
+            assert state["httpAttempts"] <= number, state
+            target = 0 if number < 3 else (2 if args.case == "A" else 3)
+            if spec["state"] == target and spec["run_attempt_count"] == number and state["httpAttempts"] == number:
+                if number < 3:
+                    assert spec["last_enqueue_time"] == expected_clock, spec
+                    actual = jobs(f"attempt-{number}-job")
+                    ids = tables["SystemIdInfo"]
+                    if (spec["schedule_requested_at"] != -1 and len(actual) == len(ids) == 1
+                            and ids[0]["work_spec_id"] == spec["id"] and ids[0]["generation"] == spec["generation"]
+                            and actual[0]["id"] == ids[0]["system_id"]):
+                        assert actual[0]["namespace"] is None
+                        return tables, actual[0], state
+                else:
+                    return tables, None, state
+            assert time.monotonic() < deadline, (spec, state)
+            time.sleep(0.1)
+
+    try:
+        with socket.socket() as probe:
+            assert probe.connect_ex(("127.0.0.1", 18081)) != 0, "Fixture18081 already owned"
+        with (evidence / "fixture.log").open("wb") as log:
+            result["fixtureCommand"] = [args.node, str(root / "fixture/server.cjs")]
+            fixture = subprocess.Popen(result["fixtureCommand"], stdout=log, stderr=subprocess.STDOUT)
+        result["fixturePid"] = fixture.pid
+        deadline = time.monotonic() + 5
+        while True:
+            assert fixture.poll() is None
+            try:
+                remote("/__state")
+                break
+            except Exception:
+                assert time.monotonic() < deadline
+                time.sleep(0.1)
+        original_network = network("initial")
+        result["networkBefore"] = original_network
+        assert original_network == ONLINE
+        assert "Success" in adb("install-candidate", "install", "-r", str(Path(args.apk).resolve()))
+        adb("wake", "shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("keyguard", "shell", "wm", "dismiss-keyguard")
+        adb("clear-logcat", "logcat", "-c")
+        adb("setup-stop", "shell", "am", "force-stop", PACKAGE)
+        assert adb("setup-clear", "shell", "pm", "clear", PACKAGE) == "Success"
+        quiet_reset()
+        assert remote("/__m10-reset", {"case": args.case})["httpAttempts"] == 0
+        result["setupOffline"] = network("setup-offline", OFFLINE)
+        adb("setup-files", "shell", "run-as", PACKAGE, "mkdir", "-p", "files")
+        # Clock T is later than ALL setup history, never the remote 2023 clock.
+        exits = adb("setup-exit-history", "shell", "dumpsys", "activity", "exit-info", PACKAGE)
+        offset = adb("setup-timezone", "shell", "date", "+%z")
+        device_now = int(adb("setup-wall-clock", "shell", "date", "+%s")) * 1000
+        history = [int(datetime.datetime.strptime(value + offset, "%Y-%m-%d %H:%M:%S.%f%z").timestamp() * 1000)
+                   for value in re.findall(r"timestamp=(\d{4}-\d\d-\d\d \d\d:\d\d:\d\d\.\d{3})", exits)]
+        initial_clock = max([device_now] + history) + 60000
+        result["clockOrigin"] = {"deviceEpochMs": device_now, "exitTimestamps": history, "T": initial_clock, "offset": offset}
+        clock("clock-initial", initial_clock)
+        extras = ("--ez", "m10FixedIdentity", "true", "--es", "m07MutationIdentity", inputs["clientMutationId"])
+        result["pidBefore"] = launch("production-launch", *extras)
+        find("Local storage ready")
+        result["firstActivity"] = resumed_activity("first-resumed-activity")
+        find("Item count: 0")
+        find("Pending uploads: 0")
+        tap("New item title")
+        adb("type-title", "shell", "input", "text", "Background%sitem")
+        tap("Add item")
+        find("Local storage ready")
+        find("Pending uploads: 1")
+        find("Background sync: active")
+        snapshot("committed-offline-ui")
+        before = database("committed-offline-items", "files/items.db")
+        row = before["pending_mutations"][0]
+        assert len(before["pending_mutations"]) == len(before["items"]) == 1
+        assert row["client_mutation_id"] == inputs["clientMutationId"] and row["payload_hash"] == inputs["payloadHash"]
+        assert json.loads(row["payload"]) == inputs["payload"] and row["dispatched"] == 0 and row["terminal_error"] is None
+        assert before["sync_metadata"] == [{"singleton": 1, "last_successful_refresh_at": None, "last_acknowledgement": None}]
+        assert before["items"][0]["id"] == "work-001" and before["items"][0]["version"] == 1
+        scheduled = database("registered-work", WORK_DB)
+        spec = work(scheduled)
+        result.update(workId=spec["id"], workInputHex=spec["input"].hex(), productionMutationInvoked=True)
+        assert spec["state"] == 0 and spec["run_attempt_count"] == 0 and spec["last_enqueue_time"] == initial_clock
+        watermark = {row["key"]: row["long_value"] for row in scheduled["Preference"]}.get("last_force_stop_ms", 0)
+        result["forceStopWatermark"] = watermark
+        if "reason=10 (USER REQUESTED)" in exits:
+            assert watermark == initial_clock and all(stamp < watermark for stamp in history)
+        state = preferences("registered-cycle")
+        assert state["status"] == "active" and state["attempts"] == 0
+        assert state["cycleId"].encode() in spec["input"]
+        result["cycleId"] = state["cycleId"]
+        result["jobBefore"] = registered("registered-jobs", scheduled)
+        # A second real Activity root takes the owned startup registration path.
+        # It runs before loss, and does not reset the cycle or invoke manual Sync.
+        assert launch("repeat-owned-startup", "-f", "0x18000000", *extras) == result["pidBefore"]
+        find("Local storage ready")
+        find("Background sync: active")
+        result["repeatedActivity"] = resumed_activity("repeat-owned-activities")
+        assert result["repeatedActivity"]["record"] != result["firstActivity"]["record"]
+        assert result["repeatedActivity"]["task"] != result["firstActivity"]["task"]
+        repeated = database("repeated-work", WORK_DB)
+        assert work(repeated)["id"] == result["workId"] and len(repeated["WorkSpec"]) == 1
+        assert preferences("repeated-cycle") == state
+        assert registered("repeated-jobs", repeated)["id"] == result["jobBefore"]["id"]
+        no_http()
+        adb("background-home", "shell", "input", "keyevent", "KEYCODE_HOME")
+        deadline = time.monotonic() + 10
+        while True:
+            activities = adb("background-activities", "shell", "dumpsys", "activity", "activities")
+            resumed = [line.strip() for line in activities.splitlines() if "mResumedActivity:" in line or "topResumedActivity=" in line]
+            if resumed and all(PACKAGE not in line for line in resumed):
+                break
+            assert time.monotonic() < deadline
+            time.sleep(0.1)
+        uid = adb("same-uid", "shell", "run-as", PACKAGE, "id", "-u")
+        process = adb("verified-process-uid", "shell", "ps", "-p", result["pidBefore"], "-o", "UID,PID,NAME")
+        assert process.splitlines()[1].split() == [uid, result["pidBefore"], PACKAGE]
+        result["sameUid"] = uid
+        result["packageBeforeLoss"] = not_stopped("package-before-loss")
+        assert network("before-loss-network") == OFFLINE
+        assert adb("immediate-before-kill", "shell", "pidof", PACKAGE) == result["pidBefore"]
+        result["killCommandIndex"] = len(commands)
+        adb("same-uid-kill9", "shell", "run-as", PACKAGE, "kill", "-9", result["pidBefore"])
+        result["processLoss"] = no_process("absent-after-signal")
+        result["packageAfterLoss"] = not_stopped("package-after-loss")
+        assert database("after-loss-items", "files/items.db") == before
+        lost = database("after-loss-work", WORK_DB)
+        assert work(lost) == work(repeated)
+        assert registered("after-loss-jobs", lost)["id"] == result["jobBefore"]["id"]
+        assert preferences("after-loss-cycle") == state
+        exits = adb("exit-history-after-loss", "shell", "dumpsys", "activity", "exit-info", PACKAGE)
+        assert re.search(r"pid=" + result["pidBefore"] + r"\b[^\n]*\n\s*process=" + re.escape(PACKAGE) + r" reason=2 .*status=9", exits)
+        result["unmetInvocation"] = run_job("unmet-job-run", result["jobBefore"], unmet=True)
+        result["unmetNoProcess"] = no_process("absent-after-unmet-job")
+        no_http()
+        assert work(database("unmet-work", WORK_DB))["run_attempt_count"] == 0
+        assert preferences("unmet-cycle") == state
+        result["networkOnline"] = network("online-without-app-entry", ONLINE)
+        first_observed = database("online-first-work", WORK_DB)
+        first_spec = work(first_observed)
+        if first_spec["state"] == 0 and first_spec["run_attempt_count"] == 0:
+            result["firstEligibleInvocation"] = run_job("first-eligible-job", result["jobBefore"])
+        else:
+            result["firstEligibleInvocation"] = {"naturalOsDispatchAlreadyObserved": True, "state": first_spec["state"], "dbAttempt": first_spec["run_attempt_count"]}
+        last_job = result["jobBefore"]
+        for number, at in ((1, initial_clock), (2, initial_clock + 10000), (3, initial_clock + 30000)):
+            if number > 1:
+                clock(f"clock-attempt-{number}", at)
+                result.setdefault("eligibleInvocations", []).append(run_job(f"eligible-job-{number}", last_job))
+            tables, next_job, remote_state = wait_attempt(number, at)
+            cycle = preferences(f"after-attempt-{number}-cycle")
+            assert cycle == {"cycleId": result["cycleId"], "attempts": number,
+                             "status": "active" if number < 3 else ("complete" if args.case == "A" else "deferred")}, cycle
+            assert remote_state["requests"][-1]["status"] == inputs["expectedStatuses"][args.case][number - 1]
+            pid = adb(f"after-attempt-{number}-pid", "shell", "pidof", PACKAGE)
+            assert re.fullmatch(r"\d+", pid) and pid != result["pidBefore"], pid
+            if number == 1:
+                result["pidAfter"] = pid
+                result["osDispatchObservedByCommandIndex"] = len(commands) - 1
+            result.setdefault("attempts", []).append({"number": number, "clock": at,
+                "dbCount": work(tables)["run_attempt_count"], "dbState": work(tables)["state"],
+                "lastEnqueueTime": work(tables)["last_enqueue_time"], "pid": pid, "cycle": cycle, "nextJob": next_job})
+            items = database(f"after-attempt-{number}-items", "files/items.db")
+            if number < 3 or args.case == "B":
+                expected = json.loads(json.dumps(before))
+                expected["pending_mutations"][0]["dispatched"] = 1
+                assert items == expected
+            if next_job is not None:
+                last_job = next_job
+        deadline = time.monotonic() + 10
+        while jobs("terminal-jobs"):
+            assert time.monotonic() < deadline, "Terminal work left an OS job registered"
+            time.sleep(0.1)
+        result["afterExhaustionProbe"] = run_job("terminal-job-probe", last_job)
+        if args.case == "B":
+            clock("clock-after-exhaustion", initial_clock + 90000)
+            result["lateExhaustionProbe"] = run_job("late-terminal-job-probe", last_job)
+            result["deferredUiPid"] = launch("ordinary-deferred-ui-launch")
+            find("Local storage ready")
+            find("Background sync: deferred")
+            find("Pending uploads: 1")
+            snapshot("deferred-ui")
+            assert preferences("deferred-ui-cycle") == {"cycleId": result["cycleId"], "attempts": 3, "status": "deferred"}
+            assert work(database("deferred-ui-work", WORK_DB))["state"] == 3
+            assert not jobs("deferred-ui-jobs")
+        final_items = database("final-items", "files/items.db")
+        if args.case == "A":
+            final = inputs["finalItem"]
+            assert final_items["items"] == [{"id": final["id"], "title": final["title"], "completed": 0,
+                                             "version": 1, "updated_at": final["updatedAt"]}]
+            assert final_items["pending_mutations"] == []
+            receipt = json.loads(final_items["sync_metadata"][0]["last_acknowledgement"])
+            assert receipt == {"clientMutationId": inputs["clientMutationId"], "payloadHash": inputs["payloadHash"], "status": 201, "result": {"item": final}}
+        else:
+            expected = json.loads(json.dumps(before)); expected["pending_mutations"][0]["dispatched"] = 1
+            assert final_items == expected
+        assert final_items["sync_metadata"][0]["last_successful_refresh_at"] is None
+        result["remoteFinal"] = remote()
+        result["mutationsFinal"] = remote("/__mutation-state")
+        assert result["remoteFinal"]["httpAttempts"] == len(result["remoteFinal"]["requests"]) == 3
+        assert [event["status"] for event in result["remoteFinal"]["requests"]] == inputs["expectedStatuses"][args.case]
+        assert all(event["method"] == "POST" and event["path"] == "/items" for event in result["remoteFinal"]["requests"])
+        assert result["remoteFinal"]["items"] == ([inputs["finalItem"]] if args.case == "A" else [])
+        mutations = result["mutationsFinal"]
+        assert {key: mutations[key] for key in ("appliedCount", "duplicateCount", "conflictCount", "hashRejectedCount")} == {
+            "appliedCount": 1 if args.case == "A" else 0, "duplicateCount": 0, "conflictCount": 0, "hashRejectedCount": 0}
+        assert len(mutations["attempts"]) == 3
+        for event in mutations["attempts"]:
+            assert event["wireBody"] == inputs["wireBody"] and event["clientMutationId"] == inputs["clientMutationId"]
+            assert event["canonical"] == inputs["canonical"] and event["actualHash"] == event["declaredHash"] == inputs["payloadHash"]
+        logs = adb("verified-logcat", "logcat", "-d", "-v", "threadtime")
+        events = [json.loads(found[1]) for line in logs.splitlines()
+                  if (found := re.search(r"\bMSEBackground\s*: (\{.*\})$", line))]
+        result["workerEvents"] = events
+        starts = [event for event in events if event["phase"] == "START"]
+        assert [event["workerAttempt"] for event in starts] == [0, 1, 2], starts
+        assert [event["clock"] for event in starts] == [initial_clock, initial_clock + 10000, initial_clock + 30000]
+        assert all(event["cycleId"] == result["cycleId"] and event["workId"] == result["workId"] for event in events)
+        assert [event["reservedAttempts"] for event in events if event["phase"] == "HTTP_RESERVED"] == [1, 2, 3]
+        assert len([event for event in events if event["phase"] == "HTTP_FINISHED"]) == 3
+        assert re.search(r"Start proc " + result["pidAfter"] + ":" + re.escape(PACKAGE) + r"/.*for service.*SystemJobService", logs)
+        assert any("WM-SystemJobService" in line and "onStartJob" in line and result["workId"] in line for line in logs.splitlines())
+        assert "TEST_DELAY_HELD" in logs
+        assert fixture.poll() is None and os.getpid() == result["hostPid"]
+        result.update(status="PASS", fixturePidAcrossLoss=fixture.pid, pendingAfter=len(final_items["pending_mutations"]),
+                      actualHttpAttempts=3, boundaryLastCommandIndex=result["osDispatchObservedByCommandIndex"])
+    except Exception as error:
+        result.update(status="FAIL", error=repr(error))
+    finally:
+        try:
+            adb("cleanup-logcat", "logcat", "-d", "-v", "threadtime")
+            adb("cleanup-stop", "shell", "am", "force-stop", PACKAGE)
+            result["pidAfterCleanup"] = no_process("cleanup-pid")
+            adb("cleanup-clock", "shell", "run-as", PACKAGE, "rm", "-f", CLOCK, CLOCK + ".next")
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
+                result["fixtureProcessAfterCleanup"] = {"exit": probe.returncode, "stdout": probe.stdout, "stderr": probe.stderr}
+                assert probe.returncode == 1 and not probe.stdout.strip()
+                with socket.socket() as port:
+                    result["fixturePortFree"] = port.connect_ex(("127.0.0.1", 18081)) != 0
+                assert result["fixturePortFree"]
+            except Exception as error:
+                if fixture.poll() is None:
+                    fixture.kill(); result["fixtureExit"] = fixture.wait(timeout=5)
+                result.update(status="FAIL", fixtureCleanupError=repr(error))
+        result.update(elapsedSeconds=time.monotonic() - started, adbCommands=len(commands))
+        save()
+        print(json.dumps(result), flush=True)
+    return 0 if result["status"] == "PASS" else 1
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/src/App.tsx b/src/App.tsx
index bd58808..17f5a4c 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -1,8 +1,9 @@
 import React, {useEffect, useRef, useState} from 'react';
-import {Button, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
+import {Button, DeviceEventEmitter, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
 import {Item} from './items';
-import {ItemMutation, ItemStore, openItemStore} from './itemStore';
+import {ItemMutation, ItemStore, openItemStore, openRuntimeItemStore} from './itemStore';
 import {ForegroundSync, SyncSession} from './sync';
+import {backgroundBridge, BackgroundState, ownsAutomaticSync, schedulePending, serializeSync} from './backgroundSync';
 
 const defaultSync = (store: ItemStore, identityPrefix?: string, testRefreshClock = false) => {
   // Android supplies this prop only behind BuildConfig.DEBUG. Real network and
@@ -38,6 +39,8 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const [pendingCount, setPendingCount] = useState<number | null>(null);
   const [identityBlocked, setIdentityBlocked] = useState(false);
   const [conflictCount, setConflictCount] = useState<number | null>(null);
+  const [background, setBackground] = useState<BackgroundState>({cycleId: null, attempts: 0, status: 'none'});
+  const [backgroundError, setBackgroundError] = useState<string | null>(null);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
@@ -47,7 +50,25 @@ export default function App({openStore = openItemStore, createSync = defaultSync
 
   useEffect(() => {
     mounted.current = true;
-    return () => {mounted.current = false;};
+    const subscription = DeviceEventEmitter.addListener('BackgroundSyncChanged', () => {
+      void serializeSync(async () => {
+        const opened = store.current;
+        if (!mounted.current || !opened) {return;}
+        const saved = await opened.read();
+        const pending = await opened.readPending();
+        const conflicts = await opened.readConflicts();
+        const state = await backgroundBridge.getState();
+        if (!mounted.current || store.current !== opened) {return;}
+        setItems(saved);
+        setPendingCount(pending.length);
+        setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
+        setConflictCount(conflicts.length);
+        setBackground(state);
+      }).catch(() => {
+        if (mounted.current) {setBackgroundError('Background state unavailable; saved changes are retained.');}
+      });
+    });
+    return () => {mounted.current = false; subscription.remove();};
   }, []);
 
   function updateEditor(next: EditorState) {
@@ -64,13 +85,20 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     setBusy(true);
     setError(null);
     // The Android override is debug-only and changes just the generated value.
-    const opening = openStore === openItemStore && testMutationIdentity
-      ? openItemStore(undefined, () => testMutationIdentity) : openStore();
-    opening.then(async opened => {
+    const opening = openStore === openItemStore
+      ? openRuntimeItemStore(testMutationIdentity ? () => testMutationIdentity : undefined) : openStore();
+    opening.then(opened => serializeSync(async () => {
+      if (!active) {return;}
       const saved = await opened.read();
       const lastRefresh = await opened.readLastSuccessfulRefresh();
       const pending = await opened.readPending();
       const conflicts = await opened.readConflicts();
+      let state = await backgroundBridge.getState();
+      if (active && state.status === 'active') {
+        // Re-enqueue an orphaned active marker using the SAME durable cycle and
+        // allowance. An exhausted/deferred cycle never starts a fresh chain.
+        state = await schedulePending(opened);
+      }
       if (active) {
         store.current = opened;
         sync.current = createSync(opened, testIdentityPrefix, testRefreshClock);
@@ -81,16 +109,17 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         setConflictCount(conflicts.length);
         setRefresh({status: 'stale'});
         setReady(true);
-        recoverPending = pending.some(operation => operation.terminalError === null);
+        setBackground(state);
+        recoverPending = pending.some(operation => operation.terminalError === null) && !ownsAutomaticSync(state);
       }
-    }).catch(reason => {
+    })).catch(reason => {
       if (active) {setError(`Could not open local database: ${String(reason.message ?? reason)}`);}
     }).finally(() => {
       if (active) {
         busyRef.current = false;
-        // Only the committed startup queue triggers recovery. Live edits remain
-        // explicit; the existing sync path owns busy/error/ACK completion.
-        if (recoverPending) {void synchronize();}
+        // Legacy/unowned queues retain M09 foreground recovery. Owned work uses
+        // WorkManager, without spending offline HTTP allowance during startup.
+        if (recoverPending) {void synchronize(false);}
         else {setBusy(false);}
       }
     });
@@ -115,15 +144,35 @@ export default function App({openStore = openItemStore, createSync = defaultSync
 
   async function mutate(action: ItemMutation): Promise<boolean> {
     if (!mounted.current || !store.current || busyRef.current) {return false;}
+    const origin = store.current;
+    const identityPrefix = sync.current?.identityPrefix;
     busyRef.current = true;
     setBusy(true);
     setError(null);
     try {
       // Every new Item can now upload, even before the first successful refresh.
       // Use the existing distinct namespace immediately; full IDs persist in SQL.
-      const saved = await store.current.mutate(action, sync.current?.identityPrefix);
-      if (mounted.current) {setItems(saved); setRefresh({status: 'stale'});}
-      return true;
+      // SQLite already commits each edit/intent atomically against ACKs. Do not
+      // hold the runtime upload lock across a late local callback: a remounted
+      // editor must still be able to read that committed state.
+      const committed = await origin.mutate(action, identityPrefix);
+      return await serializeSync(async () => {
+        let saved = committed;
+        try {if (mounted.current && store.current === origin) {saved = await origin.read();}}
+        catch { /* A failed status read cannot undo the confirmed local COMMIT. */ }
+        if (mounted.current && store.current === origin) {setItems(saved); setRefresh({status: 'stale'});}
+        try {
+          // The committed edit must register work even if its editor unmounted.
+          // Native scheduling retains an existing cycle and its durable count.
+          const state = await backgroundBridge.schedule();
+          if (mounted.current && store.current === origin) {setBackground(state); setBackgroundError(null);}
+        } catch {
+          // Local COMMIT already succeeded. A scheduler failure cannot label the
+          // edit unsaved; its durable pending intent remains available to Sync.
+          if (mounted.current && store.current === origin) {setBackgroundError('Background scheduling failed; saved changes are retained.');}
+        }
+        return true;
+      });
     } catch (reason) {
       if (mounted.current) {setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);}
       return false;
@@ -134,13 +183,20 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     }
   }
 
-  async function synchronize() {
+  async function synchronize(manual: boolean) {
     if (!mounted.current || !store.current || !sync.current || busyRef.current) {return;}
     busyRef.current = true;
     setBusy(true);
     setRefresh({status: 'refreshing'});
     try {
-      await sync.current.synchronize();
+      const performed = await serializeSync(async () => {
+        if (!mounted.current) {return false;}
+        if (manual) {setBackground(await backgroundBridge.prepareManual());}
+        else if (ownsAutomaticSync(await backgroundBridge.getState())) {return false;}
+        await sync.current!.synchronize();
+        return true;
+      });
+      if (!performed) {if (mounted.current) {setRefresh({status: 'stale'});} return;}
       if (!mounted.current) {return;}
       const saved = await store.current.read();
       const lastRefresh = await store.current.readLastSuccessfulRefresh();
@@ -160,6 +216,13 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         if (mounted.current) {setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});}
       }
     } finally {
+      try {
+        const state = await serializeSync(() => mounted.current
+          ? schedulePending(store.current!) : backgroundBridge.getState());
+        if (mounted.current) {setBackground(state);}
+      } catch {
+        if (mounted.current) {setBackgroundError('Background scheduling failed; saved changes are retained.');}
+      }
       await reloadPending();
       busyRef.current = false;
       if (mounted.current) {setBusy(false);}
@@ -205,9 +268,16 @@ export default function App({openStore = openItemStore, createSync = defaultSync
           Conflicts preserved: {conflictCount}. Canonical state wins after refresh. Original attempts are saved and will not retry. Edit a current Item to make a new attempt.
         </Text>}
         {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
+        <Text accessibilityLabel={`Background sync: ${background.status}`}>
+          {background.status === 'deferred'
+            ? `Background sync deferred after ${background.attempts} reserved attempts. Saved edits remain; use Synchronize to retry.`
+            : background.status === 'active' ? 'Background sync queued; Android controls timing and connectivity.'
+              : 'No active background upload cycle.'}
+        </Text>
+        {backgroundError !== null && <Text accessibilityRole="alert">{backgroundError}</Text>}
       </>}
-      <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
-      <Text>Startup resumes saved uploads. Sync uploads edits in order, then refreshes Items.</Text>
+      <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={() => void synchronize(true)} />
+      <Text>Saved edits can upload through Android background work. Synchronize uploads in order, then refreshes Items.</Text>
       <TextInput
         accessibilityLabel={editingId === null ? 'New item title' : 'Edit item title'}
         placeholder="Item title"
diff --git a/src/backgroundSync.ts b/src/backgroundSync.ts
new file mode 100644
index 0000000..1a29104
--- /dev/null
+++ b/src/backgroundSync.ts
@@ -0,0 +1,93 @@
+import {DeviceEventEmitter, NativeModules} from 'react-native';
+import {ItemStore, openRuntimeItemStore} from './itemStore';
+import {ForegroundSync, JsonRequest} from './sync';
+
+export type BackgroundState = {cycleId: string | null; attempts: number; status: 'none' | 'active' | 'deferred' | 'complete'};
+export interface BackgroundBridge {
+  getState(): Promise<BackgroundState>;
+  schedule(): Promise<BackgroundState>;
+  prepareManual(): Promise<BackgroundState>;
+  isActive(token: string): Promise<boolean>;
+  reserve(token: string): Promise<boolean>;
+  requestFinished(token: string): Promise<void>;
+  complete(token: string, outcome: 'success' | 'retry' | 'failure'): Promise<boolean>;
+}
+export const backgroundBridge: BackgroundBridge = NativeModules.BackgroundSync;
+
+// One JS runtime serves both the Activity and WorkManager. Keep the whole
+// upload/ACK operation serialized; scheduling waits only for durable enqueue.
+let previous: Promise<unknown> = Promise.resolve();
+export function serializeSync<T>(operation: () => Promise<T>): Promise<T> {
+  const result = previous.then(operation, operation);
+  previous = result.catch(() => undefined);
+  return result;
+}
+
+export const ownsAutomaticSync = (state: BackgroundState) => state.status === 'active' || state.status === 'deferred';
+
+export async function schedulePending(store: ItemStore, bridge = backgroundBridge): Promise<BackgroundState> {
+  const pending = await store.readPending();
+  return pending[0]?.terminalError === null ? bridge.schedule() : bridge.getState();
+}
+
+export async function runBackgroundTask(task: {token: string}, bridge = backgroundBridge,
+  openStore: () => Promise<ItemStore> = openRuntimeItemStore,
+  request: JsonRequest = (url, options) => fetch(url, options)): Promise<void> {
+  const controller = new AbortController();
+  const cancellation = DeviceEventEmitter.addListener('BackgroundSyncCancelled', (event: {token: string}) => {
+    if (event.token === task.token) {controller.abort();}
+  });
+  try {
+    await serializeSync(async () => {
+      if (controller.signal.aborted || !await bridge.isActive(task.token)) {return;}
+      let outcome: 'success' | 'retry' | 'failure' = 'failure';
+      let store: ItemStore | undefined;
+      try {
+        store = await openStore();
+        const guardedRequest: JsonRequest = async (url, options) => {
+          let reservationMayExist = false;
+          try {
+            if (controller.signal.aborted) {throw new Error('Background upload stopped');}
+            // A rejected bridge call may still have committed its reservation.
+            // Always settle that token before allowing native timeout/retry.
+            reservationMayExist = true;
+            if (!await bridge.reserve(task.token)) {
+              reservationMayExist = false;
+              throw new Error('Automatic upload allowance unavailable');
+            }
+            if (controller.signal.aborted || !await bridge.isActive(task.token)) {
+              throw new Error('Background upload stopped');
+            }
+            const response = await request(url, {...options, signal: controller.signal});
+            // Keep the reservation in flight through body consumption, not
+            // merely response headers, and reject a cancelled late body.
+            const body = await response.json();
+            if (controller.signal.aborted) {throw new Error('Background upload stopped');}
+            return {status: response.status, json: async () => body};
+          } finally {
+            // Native timeout cannot retry while its previous fetch is still live.
+            if (reservationMayExist) {await bridge.requestFinished(task.token);}
+          }
+        };
+        await new ForegroundSync(store, undefined, guardedRequest).uploadPending();
+        outcome = 'success'; // Only after the existing native ACK transaction.
+      } catch {
+        if (!controller.signal.aborted) {
+          try {
+            const pending = await store?.readPending();
+            const state = await bridge.getState();
+            outcome = state.attempts > 0 && pending?.[0]?.terminalError !== 'identity_conflict' ? 'retry' : 'failure';
+          } catch {outcome = 'failure';}
+        }
+      }
+      // Keep cycle completion with the drain, so a new edit cannot be enqueued
+      // into a cycle that is already about to finish. This awaits only the
+      // native result acknowledgment, never WorkManager execution completion.
+      if (!controller.signal.aborted) {await bridge.complete(task.token, outcome);}
+    });
+  } finally {
+    cancellation.remove();
+  }
+  // Headless task finish is only bookkeeping. Explicit correlated completion
+  // supplies the WorkManager result; stale/stopped tokens are rejected natively.
+}
diff --git a/src/itemStore.ts b/src/itemStore.ts
index d6a3944..7d60116 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -256,10 +256,19 @@ class SqliteItemStore implements ItemStore {
             [first.sequence, first.clientMutationId, first.payloadHash]);
           if ('item' in result) {
             const accepted = {...result.item, deleted: false, item: result.item};
+            const successor = pending.slice(1).find(candidate => candidate.itemId === first.itemId);
             readVersions(tx, versions => {
-              if (acceptsVersion(versions.get(accepted.id), accepted)) {writeVersion(tx, accepted);}
+              if (acceptsVersion(versions.get(accepted.id), accepted)) {
+                writeVersion(tx, accepted);
+                // Upload-only background work has no trailing collection GET.
+                // Promote this ACK without overwriting any later local intent.
+                if (!successor) {
+                  const row = itemToRow(result.item);
+                  tx.executeSql('UPDATE items SET title = ?, completed = ?, version = ?, updated_at = ? WHERE id = ?',
+                    [row.title, row.completed, row.version, row.updated_at, row.id]);
+                }
+              }
             });
-            const successor = pending.slice(1).find(candidate => candidate.itemId === first.itemId);
             if (successor && successor.kind !== 'create' && !successor.dispatched && successor.terminalError === null) {
               // Only this acknowledged OWN predecessor can prepare a never-sent
               // successor. External GET/conflict versions never rebase a queue.
@@ -476,7 +485,7 @@ function upgradeFromFour(tx: SQLTransaction) {
   });
 }
 
-export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMutationIdentity): Promise<ItemStore> {
+async function initializeDatabase(name: string, makeIdentity: () => string): Promise<WebsqlDatabase> {
   const database = SQLite.openDatabase(name);
   await new Promise<void>((resolve, reject) => {
     database.transaction(tx => {
@@ -524,5 +533,22 @@ export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMuta
       });
     }, reject, resolve);
   });
-  return new SqliteItemStore(database, makeIdentity);
+  return database;
+}
+
+export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMutationIdentity): Promise<ItemStore> {
+  return new SqliteItemStore(await initializeDatabase(name, makeIdentity), makeIdentity);
+}
+
+// One native database needs one WebSQL transaction queue per JS runtime.
+// Keep each caller's identity source immutable, including debug Activity inputs.
+// Injected/test databases still use the independent opener above.
+let runtimeDatabase: Promise<WebsqlDatabase> | undefined;
+export async function openRuntimeItemStore(makeIdentity = newMutationIdentity): Promise<ItemStore> {
+  if (!runtimeDatabase) {
+    const opening = initializeDatabase(DATABASE_NAME, makeIdentity);
+    runtimeDatabase = opening;
+    void opening.catch(() => {if (runtimeDatabase === opening) {runtimeDatabase = undefined;}});
+  }
+  return new SqliteItemStore(await runtimeDatabase, makeIdentity);
 }
diff --git a/src/sync.ts b/src/sync.ts
index 3188251..f82cea8 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -5,7 +5,7 @@ import {mutationTarget} from './mutationProtocol';
 export const FIXTURE_URL = 'http://10.0.2.2:18081';
 
 export type JsonRequest = (url: string, options: {
-  method: string; headers: Record<string, string>; body?: string;
+  method: string; headers: Record<string, string>; body?: string; signal?: AbortSignal;
 }) => Promise<{status: number; json(): Promise<unknown>}>;
 
 export interface SyncSession {
@@ -77,6 +77,22 @@ export class ForegroundSync implements SyncSession {
   }
 
   async synchronize(): Promise<void> {
+    await this.uploadPending();
+
+    const response = await this.exchange('GET', '/items', 200) as {items?: unknown; tombstones?: unknown};
+    if (!Array.isArray(response.items) || (response.tombstones !== undefined && !Array.isArray(response.tombstones))) {
+      throw new Error('Invalid remote snapshot');
+    }
+    const snapshot = response.items.map(remoteItem);
+    const tombstones = ((response.tombstones ?? []) as unknown[]).map(remoteTombstone);
+    if (new Set([...snapshot, ...tombstones].map(item => item.id)).size !== snapshot.length + tombstones.length) {
+      throw new Error('Duplicate Item or tombstone in remote snapshot');
+    }
+    await this.store.replaceSnapshot(snapshot, this.now(), tombstones);
+    this.refreshed = true;
+  }
+
+  async uploadPending(): Promise<void> {
     // Recover ordered upload intent from SQLite, including on the first sync
     // after process restart. Each edit is sent separately, without coalescing.
     for (;;) {
@@ -100,17 +116,5 @@ export class ForegroundSync implements SyncSession {
       }
       await this.store.acknowledge(operation, result);
     }
-
-    const response = await this.exchange('GET', '/items', 200) as {items?: unknown; tombstones?: unknown};
-    if (!Array.isArray(response.items) || (response.tombstones !== undefined && !Array.isArray(response.tombstones))) {
-      throw new Error('Invalid remote snapshot');
-    }
-    const snapshot = response.items.map(remoteItem);
-    const tombstones = ((response.tombstones ?? []) as unknown[]).map(remoteTombstone);
-    if (new Set([...snapshot, ...tombstones].map(item => item.id)).size !== snapshot.length + tombstones.length) {
-      throw new Error('Duplicate Item or tombstone in remote snapshot');
-    }
-    await this.store.replaceSnapshot(snapshot, this.now(), tombstones);
-    this.refreshed = true;
   }
 }
diff --git a/verification/M10.md b/verification/M10.md
index 5ee724a..5c60096 100644
--- a/verification/M10.md
+++ b/verification/M10.md
@@ -2,8 +2,8 @@
 
 - Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: verified M09 `ed35bc3b686f52379b7cd16728171191a1386785`.
-- Status: **unchanged-M09 prerequisite limitation accepted by root**; product implementation and final A/B are NOT_RUN.
-- Attempt2; repair1/2 used. Fixed-case budget: A2/3, B0/3. Root owns remaining starts; no baseline rerun.
+- Status: **root runtime/source/artifact verification PASS**; final committed-byte/history audit and tag remain root-owned.
+- Attempt2; repair1/2 used. Final fixed-case budget: A3/3, B1/3. Both baselines and final A are charged; no further A invocation is available.
 - Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/`.
 
 ## Frozen baseline and preserved failure
@@ -16,4 +16,25 @@ The [first baseline](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phas
 
 Root's [repaired invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/repair1-baseline-android-01/result.json) returned **LIMITATION_REPRODUCED** (33.819s,111 adb commands). Background PID26233 became absent after same-UID10238 SIGKILL; package remained `stopped=false`. Four native SQLite snapshots retain the exact work-001/m10-create-001 envelope: seed dispatched0, M09 offline startup dispatched1, then unchanged after loss and10s online. No app entrypoint, OS job, WorkDatabase or HTTP request followed loss. This does not claim OS dispatch or final A acceptance. [Root native/process audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-repair1-baseline-audit.json).
 
-Fixture73777 survived loss, exited0 and was directly absent; port18081 was free, app absent, and network0/1/1 with active129 restored. [Root cleanup audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-repair1-cleanup-audit.json). Both baselines were root-owned and charged to A; B has not executed. This commit contains only accepted baseline support and this ledger; product work awaits root's blob check.
+Fixture73777 survived loss, exited0 and was directly absent; port18081 was free, app absent, and network0/1/1 with active129 restored. [Root cleanup audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-repair1-cleanup-audit.json). Both baselines were root-owned and charged to A; B was still unstarted at preserved baseline-only commit `7bd56fa7b6a12c8ad9a407845f6f5fc66b171680`.
+
+## Frozen candidate and host verification
+
+The [candidate manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/candidate-manifest.json), SHA256 `7c9f27e4fca02237ff03b8a132cf05923055ff578efa6ed445ad33c935724a20`, records exact separate A/B argv, inputs, all68 source files, APKs and retained raw invocations. App SHA256 `7f8c1110bc38ca195d1572d6b419a9e5a3dc97cb5441df208aa70900fe8b5c27`; test APK `fcf6931c379bdbb2bcf6ec02c280e3f04b09b3de8068a77915850401856f3043`.
+
+Full Jest106/106, typecheck04, native JVM2/2 and build02 passed. Earlier Jest105/106, typecheck02 and build01 failures remain preserved, including the corrected failed-GET expectation after atomic canonical ACK promotion and explicit compile dependency. Root checked all67 pre-build files and independently disassembled the native classes/bundle; resolved and packaged WorkManager is2.9.1. No unchanged rebuild or owner device run occurred. [Host/source review](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-prefreeze-source-host-review.json); [artifact audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-artifact-audit.json).
+
+## Root actual Android verification
+
+Production UI created the fixed `work-001`/`m10-create-001` intent and registered real CONNECTED work before same-UID10238 SIGKILL. Root audited39 native Item/WorkDatabase snapshots, actual default-namespace nonforced OS jobs, immutable identity/hash, HTTP0 while constrained, and Worker attempts0/1/2 at T/T+10000/T+30000 after each committed retry. No app entrypoint or queue injection occurred between process loss and OS dispatch.
+
+| Case | Actual PID boundary | HTTP results | Final result | Raw audit |
+| --- | --- | --- | --- | --- |
+| A3 | 30474→absent→31214 | 503,503,201 | One apply, pending0; canonical Background item/false/v1/t1700000700000 | [21 native DBs;40.592s/191 adb](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-final-A-audit.json) |
+| B1 | 31978→absent→32634 | 503,503,503 | Deferred/pending1; later OS probe and ordinary UI cause no fourth HTTP/new cycle | [18 native DBs;49.865s/204 adb](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-final-B-audit.json) |
+
+Background ACK/dequeue/canonical promotion completed without GET or refresh-time advancement. Fixtures22445/26152 exited0 and were directly absent; port18081 was free, app absent, and network0/1/1 with active135/137 restored. [A cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-final-A-cleanup.json); [B cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-final-B-cleanup.json).
+
+New regressions were explicitly **OFFLINE under the M10 CONNECTED constraint**, with production scheduling enabled: [M08 native lifecycle](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-lifecycle-audit.json) passed1 test/6 native DBs (16.517s/34 adb), retaining draft/selection across background and real recreation in PID1484, then saving exactly once with pending1/dispatched0/HTTP0; [unchanged M01 CRUD](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-native-crud-01/result.json) passed1 real test. Root restored online0/1/1 with active network and app absent. Historical online M08/M01 and M05/M06/M07/M09 device cases were retained, **not rerun**; current host tests cover shared protocol/storage and unowned M09 startup.
+
+Root's read-only lifecycle auditor needed old-result control-count/path corrections; [both errors are preserved](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-lifecycle-auditor-adaptation.json). Runtime, harness and artifacts passed unchanged without rerun or repair-count change. [Final root runtime record](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M10/main-runtime-verification.json). Only this ledger and TRACK metadata changed after testing; no phase-2 work, push integration or Git push was performed.
