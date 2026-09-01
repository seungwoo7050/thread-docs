# M14 — 느린 Network와 Sync Backpressure

## `feat: cancel foreground uploads on network loss`

diff --git a/TRACK.md b/TRACK.md
index a0d84b3..e97d99e 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -252,6 +252,29 @@ not rerun; current host regressions cover those shared semantics. No phase-2
 feature or push integration is included. See `verification/M10.md` and its frozen
 manifest for the separate, budgeted A/B commands and complete raw evidence.
 
+## M14: overlapping requests and foreground cancellation (phase-1)
+
+Foreground synchronization now owns an AbortController and an Android default
+network observer before entering the existing serial upload queue. Real network
+loss or screen disposal cancels queued/active work; late response bodies cannot
+start acknowledgment, conflict or snapshot commits after cancellation. Cleanup
+removes the exact observer and subscription. WorkManager coordination, its
+three-attempt allowance, durable envelopes and storage transactions remain intact.
+
+The existing HTTP factory now owns its actual default connection pool. An owned
+network-loss callback evicts idle connections, with another idle-only sweep when
+that loss-marked observer stops. This prevents reuse of the stale idle socket
+observed on Android; it adds no HTTP retry or universal active-socket guarantee.
+The matching requery dependency added to androidTest is observation support only.
+
+Main verified twelve real offline UI creates, four native scheduling requests
+plus one accepted foreground request, exactly three durable acknowledgments
+before cancellation, and convergence after one foreground reconnect request.
+Normal background work also resumed before that reconnect click and remained
+enabled. See [M14 verification](/private/tmp/mobile-systems-evolution-ed7baa2/react-native/verification/M14.md)
+for measured concurrency, preserved failures and the unchanged earlier evidence.
+No M15 or phase-2 implementation is included.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 8dbc283..ae90004 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -5,7 +5,7 @@ import RootApp, {createEditorMemory} from '../src/App';
 import {openItemStore} from '../src/itemStore';
 import {closeDatabases, failNextSql} from './sqliteNative';
 import {ForegroundSync, JsonRequest} from '../src/sync';
-import {BackgroundState, serializeSync} from '../src/backgroundSync';
+import {BackgroundState, observeForegroundSync, serializeSync} from '../src/backgroundSync';
 
 const saved = () => waitFor(() => expect(screen.getByLabelText('Local storage ready')).toBeTruthy());
 let editorMemory: ReturnType<typeof createEditorMemory>;
@@ -771,3 +771,132 @@ test('M10 background listener cleanup and stale callback guards leave a remounte
   current.unmount();
   listen.mockRestore();
 });
+
+test.each(['initial offline', 'queued loss'] as const)('M14 %s cancels foreground ownership before manual allowance reset', async phase => {
+  const store = await openItemStore(undefined, () => 'unit-queued-cancel');
+  await store.mutate({type: 'create', title: 'Queued unit', now: 1700000000000}, 'unit-queued');
+  const pending = await store.readPending();
+  const deferred: BackgroundState = {cycleId: 'retained-cycle', attempts: 3, status: 'deferred'};
+  NativeModules.BackgroundSync.getState.mockResolvedValue(deferred);
+  NativeModules.BackgroundSync.schedule.mockResolvedValue(deferred);
+  NativeModules.BackgroundSync.observeForegroundNetwork.mockResolvedValue(phase !== 'initial offline');
+  const synchronize = jest.fn(async () => {});
+  render(<App openStore={async () => store}
+    createSync={() => ({initialized: false, identityPrefix: 'unit', synchronize})} />);
+  await saved();
+  let release!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  const owner = serializeSync(() => held);
+  let token!: string;
+  try {
+    fireEvent.press(screen.getByLabelText('Synchronize'));
+    await waitFor(() => expect(NativeModules.BackgroundSync.observeForegroundNetwork).toHaveBeenCalledTimes(1));
+    token = NativeModules.BackgroundSync.observeForegroundNetwork.mock.calls[0][0];
+    if (phase === 'queued loss') {
+      await act(async () => {DeviceEventEmitter.emit('ForegroundSyncOffline', {token});});
+    }
+    expect(synchronize).not.toHaveBeenCalled();
+  } finally {
+    await act(async () => {release(); await owner;});
+  }
+  await saved();
+  expect(screen.getByLabelText('Sync status: error')).toBeTruthy();
+  expect(screen.getByLabelText('Background sync: deferred')).toBeTruthy();
+  expect(NativeModules.BackgroundSync.prepareManual).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.reserve).not.toHaveBeenCalled();
+  expect(synchronize).not.toHaveBeenCalled();
+  expect(NativeModules.BackgroundSync.stopObservingForegroundNetwork).toHaveBeenCalledWith(token);
+  expect(await store.readPending()).toEqual(pending);
+});
+
+test('M14 foreground loss settles UI and a late old event cannot cancel fresh reconnect', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed], 1700000000000);
+  const signals: AbortSignal[] = [];
+  let release!: () => void;
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn(async (_address, options) => {
+    const signal = options.signal!;
+    signals.push(signal);
+    if (signals.length === 1) {
+      return new Promise<never>((_resolve, reject) => {
+        signal.addEventListener('abort', () => reject(new Error('Foreground request aborted')), {once: true});
+      });
+    }
+    await new Promise<void>(resolve => {release = resolve;});
+    return {status: 200, json: async () => ({items: [m08Input.seed]})};
+  });
+  render(<App openStore={async () => store}
+    createSync={() => new ForegroundSync(store, 'http://unit-foreground', transport)} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(transport).toHaveBeenCalledTimes(1));
+  const oldToken = NativeModules.BackgroundSync.observeForegroundNetwork.mock.calls[0][0];
+  await act(async () => {DeviceEventEmitter.emit('ForegroundSyncOffline', {token: oldToken});});
+  await saved();
+  expect(signals[0].aborted).toBe(true);
+  expect(screen.getByLabelText('Sync status: error')).toBeTruthy();
+  expect(screen.getByLabelText('Synchronize').props.accessibilityState.disabled).toBe(false);
+  expect(NativeModules.BackgroundSync.stopObservingForegroundNetwork).toHaveBeenCalledWith(oldToken);
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(transport).toHaveBeenCalledTimes(2));
+  const newToken = NativeModules.BackgroundSync.observeForegroundNetwork.mock.calls[1][0];
+  expect(newToken).not.toBe(oldToken);
+  await act(async () => {DeviceEventEmitter.emit('ForegroundSyncOffline', {token: oldToken});});
+  expect(signals[1].aborted).toBe(false);
+  await act(async () => {release();});
+  await saved();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(NativeModules.BackgroundSync.stopObservingForegroundNetwork.mock.calls).toEqual([[oldToken], [newToken]]);
+});
+
+test('M14 unmount aborts foreground transport and leaves a remounted draft untouched', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  let signal!: AbortSignal;
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn(async (_address, options) => {
+    signal = options.signal!;
+    return new Promise<never>((_resolve, reject) => {
+      signal.addEventListener('abort', () => reject(new Error('Old root aborted')), {once: true});
+    });
+  });
+  const first = render(<App openStore={async () => store}
+    createSync={() => new ForegroundSync(store, 'http://old-root', transport)} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Keep current draft');
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(transport).toHaveBeenCalledTimes(1));
+  const token = NativeModules.BackgroundSync.observeForegroundNetwork.mock.calls[0][0];
+  first.unmount();
+  expect(signal.aborted).toBe(true);
+  await waitFor(() => expect(NativeModules.BackgroundSync.stopObservingForegroundNetwork).toHaveBeenCalledWith(token));
+  render(<App openStore={async () => store} />);
+  await saved();
+  await act(async () => {DeviceEventEmitter.emit('ForegroundSyncOffline', {token});});
+  expect(screen.getByLabelText('Edit item title').props.value).toBe('Keep current draft');
+  expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+  expect(await store.readPending()).toEqual([]);
+});
+
+test.each(['resolve', 'reject'] as const)('M14 early observer disposal removes its listener and stops exact token after registration %s', async outcome => {
+  let resolve!: (connected: boolean) => void;
+  let reject!: (error: Error) => void;
+  const registered = new Promise<boolean>((yes, no) => {resolve = yes; reject = no;});
+  const bridge = {observeForegroundNetwork: jest.fn((_token: string) => registered),
+    stopObservingForegroundNetwork: jest.fn(async (_token: string) => {})};
+  const listen = jest.spyOn(DeviceEventEmitter, 'addListener');
+  try {
+    const observation = observeForegroundSync(bridge);
+    const remove = jest.spyOn(listen.mock.results[0].value, 'remove');
+    const cleanup = observation.dispose();
+    expect(observation.dispose()).toBe(cleanup);
+    expect(remove).toHaveBeenCalledTimes(1);
+    expect(observation.controller.signal.aborted).toBe(true);
+    await Promise.resolve();
+    expect(bridge.observeForegroundNetwork).toHaveBeenCalledTimes(1);
+    expect(bridge.stopObservingForegroundNetwork).not.toHaveBeenCalled();
+    if (outcome === 'resolve') {resolve(true);} else {reject(new Error('Registration reply failed'));}
+    await cleanup;
+    expect(bridge.stopObservingForegroundNetwork.mock.calls).toEqual(bridge.observeForegroundNetwork.mock.calls);
+  } finally {listen.mockRestore();}
+});
diff --git a/__tests__/sqlite.setup.ts b/__tests__/sqlite.setup.ts
index 2995593..9a6d9ec 100644
--- a/__tests__/sqlite.setup.ts
+++ b/__tests__/sqlite.setup.ts
@@ -15,6 +15,7 @@ const unowned = {cycleId: null, attempts: 0, status: 'none'};
 NativeModules.BackgroundSync = {
   getState: jest.fn(), schedule: jest.fn(), prepareManual: jest.fn(),
   isActive: jest.fn(), reserve: jest.fn(), requestFinished: jest.fn(), complete: jest.fn(),
+  observeForegroundNetwork: jest.fn(), stopObservingForegroundNetwork: jest.fn(),
 };
 beforeEach(() => {
   for (const method of ['getState', 'schedule', 'prepareManual']) {
@@ -23,6 +24,8 @@ beforeEach(() => {
   for (const method of ['isActive', 'reserve', 'requestFinished', 'complete']) {
     NativeModules.BackgroundSync[method].mockReset().mockResolvedValue(false);
   }
+  NativeModules.BackgroundSync.observeForegroundNetwork.mockReset().mockResolvedValue(true);
+  NativeModules.BackgroundSync.stopObservingForegroundNetwork.mockReset().mockResolvedValue(undefined);
 });
 beforeEach(resetSqlite);
 afterEach(cleanupSqlite);
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 47af3e8..ed225d0 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -10,9 +10,14 @@ import {Item} from '../src/items';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
 
 type Trace = {method: string; path: string; body: unknown; status: number; response: unknown};
+type PressureRequest = {requestId: number; method: string; path: string; isMutation: boolean;
+  receivedAt: number; scheduledAt?: number; headersAt?: number; appliedAt?: number | null;
+  finishedAt: number | null; disconnectedAt: number | null; endedAt: number | null; delayMs?: number};
+type PressureState = {delayMs: number; inFlightMutations: number; peakInFlightMutations: number; requests: PressureRequest[]};
 const {createFixture} = require('../fixture/server.cjs') as {createFixture(): {
   server: Server; reset(): void; state(): {items: unknown[]; tombstones?: unknown[]; nextTimestamp: number; requests: Trace[]};
   mutationState(): {appliedCount: number; duplicateCount: number; conflictCount: number; hashRejectedCount: number; attempts: unknown[]};
+  pressureState(): PressureState | null;
 }};
 const fixture = createFixture();
 const url = 'http://127.0.0.1:18081';
@@ -681,6 +686,219 @@ test('M07 schema4 keeps original unversioned intent blocked and never infers can
   }
 });
 
+async function waitPressure(predicate: (state: PressureState) => boolean): Promise<PressureState> {
+  const deadline = Date.now() + 2000;
+  for (;;) {
+    const state = fixture.pressureState();
+    if (state && predicate(state)) {return state;}
+    if (Date.now() >= deadline) {throw new Error('Small fixture observation timed out');}
+    await new Promise(resolve => setTimeout(resolve, 5));
+  }
+}
+
+test('M14 fixture delays two small concurrent requests independently and measures full response lifetimes', async () => {
+  await request(`${url}/__m14-reset`, {method: 'POST', headers: {}});
+  const payloads = ['a', 'b'].map(suffix => ({id: `unit-pressure-${suffix}`, title: `Unit ${suffix}`, completed: false}));
+  const responses = Promise.all(payloads.map(payload => request(`${url}/items`, {
+    method: 'POST', headers: {'Content-Type': 'application/json'},
+    body: JSON.stringify({...payload, clientMutationId: `unit-${payload.id}`, payloadHash: mutationHash('POST', '/items', payload)}),
+  })));
+  const overlapping = await waitPressure(state => state.inFlightMutations === 2);
+  expect(overlapping.peakInFlightMutations).toBe(2);
+  expect(overlapping.requests.every(event => event.finishedAt === null)).toBe(true);
+  expect((await responses).map(response => response.status)).toEqual([201, 201]);
+  const settled = await waitPressure(state => state.inFlightMutations === 0);
+  expect(settled.delayMs).toBe(500);
+  expect(settled.requests).toHaveLength(2);
+  for (const event of settled.requests) {
+    expect(event.delayMs).toBe(500);
+    expect(event.headersAt).toBeGreaterThanOrEqual(event.scheduledAt! + 499);
+    expect(event.endedAt).toBeGreaterThanOrEqual(event.headersAt!);
+    expect(event.finishedAt).not.toBeNull();
+  }
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 2, duplicateCount: 0});
+  console.info('M14_SMALL_FIXTURE_OVERLAP', JSON.stringify(settled));
+});
+
+test('M14 coordinator settles an aborted single intent before foreground identity replay', async () => {
+  await request(`${url}/__m14-reset`, {method: 'POST', headers: {}});
+  const store = await openItemStore(undefined, () => 'unit-pressure-cancel');
+  await store.mutate({type: 'create', title: 'Unit cancel', now: 1700000999000}, 'unit-pressure');
+  const original = await store.readPending();
+  const bridge = backgroundContract();
+  const order: string[] = [];
+  bridge.requestFinished.mockImplementation(async () => {order.push('transport-settled');});
+  let active = 0;
+  let peak = 0;
+  const transport: JsonRequest = async (address, options) => {
+    active += 1; peak = Math.max(peak, active);
+    try {return await request(address.replace(FIXTURE_URL, url), options);}
+    finally {active -= 1;}
+  };
+  const running = runBackgroundTask({token: 'unit-pressure-token'}, bridge, async () => store, transport);
+  await waitPressure(state => state.requests[0]?.appliedAt != null && state.inFlightMutations === 1);
+  let beforeRecovery: PendingMutation[] = [];
+  const recovery = serializeSync(async () => {
+    order.push('foreground-entered');
+    beforeRecovery = await store.readPending();
+    await new ForegroundSync(store, url, transport).synchronize();
+  });
+  DeviceEventEmitter.emit('BackgroundSyncCancelled', {token: 'unit-pressure-token'});
+  await Promise.all([running, recovery]);
+  expect(order).toEqual(['transport-settled', 'foreground-entered']);
+  expect(beforeRecovery).toEqual(original.map(operation => ({...operation, dispatched: true})));
+  expect(bridge.complete).not.toHaveBeenCalled();
+  expect(active).toBe(0);
+  expect(peak).toBe(1);
+  expect(await store.readPending()).toEqual([]);
+  const settled = await waitPressure(state => state.inFlightMutations === 0);
+  expect(settled.peakInFlightMutations).toBeLessThanOrEqual(2);
+  expect(settled.requests[0].finishedAt).toBeNull();
+  expect(settled.requests[0].disconnectedAt).not.toBeNull();
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 1});
+  expect(await store.read()).toEqual([{id: 'unit-pressure-001', title: 'Unit cancel', completed: false,
+    version: 1, updatedAt: 1700001000000}]);
+  console.info('M14_SMALL_CANCEL_REPLAY', JSON.stringify({order, beforeRecovery, peak, fixture: settled}));
+});
+
+test('M14 a cancelled prepare cannot send, while an entered ACK may finish without sending its successor', async () => {
+  let identity = 0;
+  const store = await openItemStore(undefined, () => `unit-foreground-${++identity}`);
+  await store.mutate({type: 'create', title: 'First unit', now: 1700000000000}, 'unit-fg');
+  await store.mutate({type: 'create', title: 'Second unit', now: 1700000001000}, 'unit-fg');
+  const original = await store.readPending();
+  const controller = new AbortController();
+  const prepare = store.prepareNext.bind(store);
+  const preparing = jest.spyOn(store, 'prepareNext').mockImplementation(async () => {
+    const next = await prepare(); controller.abort(); return next;
+  });
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn();
+  await expect(new ForegroundSync(store, url, transport).uploadPending(controller.signal)).rejects.toThrow('Synchronization cancelled');
+  expect(transport).not.toHaveBeenCalled();
+  expect(await store.readPending()).toEqual([{...original[0], dispatched: true}, original[1]]);
+  preparing.mockRestore();
+
+  const reconnect = new AbortController();
+  const acknowledge = store.acknowledge.bind(store);
+  const acknowledging = jest.spyOn(store, 'acknowledge').mockImplementation(async (...args) => {
+    reconnect.abort(); // Entry is already authorized; finish the atomic ACK.
+    await acknowledge(...args);
+  });
+  const canonical = {id: 'unit-fg-001', title: 'First unit', completed: false, version: 1, updatedAt: 1700000100000};
+  transport.mockResolvedValue({status: 201, json: async () => ({item: canonical})});
+  await expect(new ForegroundSync(store, url, transport).uploadPending(reconnect.signal)).rejects.toThrow('Synchronization cancelled');
+  expect(transport).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toEqual([original[1]]);
+  expect((await store.read())[0]).toEqual(canonical);
+  expect(JSON.parse(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()!.last_acknowledgement as string)
+    .clientMutationId).toBe(original[0].clientMutationId);
+  acknowledging.mockRestore();
+});
+
+test.each(['headers', 'body'] as const)('M14 foreground rejects cancelled late %s without ACK or another send', async phase => {
+  const store = await openItemStore(undefined, () => 'unit-foreground-late');
+  await store.mutate({type: 'create', title: 'Late unit', now: 1700000000000}, 'unit-late');
+  const pending = await store.readPending();
+  const original = await store.read();
+  const controller = new AbortController();
+  let release!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  let entered!: () => void;
+  const waiting = new Promise<void>(resolve => {entered = resolve;});
+  const transport: jest.MockedFunction<JsonRequest> = jest.fn(async (_address, options) => {
+    expect(options.signal).toBe(controller.signal);
+    if (phase === 'headers') {entered(); await held;}
+    return {status: 201, json: async () => {
+      if (phase === 'body') {entered(); await held;}
+      return {item: {...original[0], updatedAt: 1700000100000}};
+    }};
+  });
+  const result = expect(new ForegroundSync(store, url, transport).synchronize(controller.signal))
+    .rejects.toThrow('Synchronization cancelled');
+  await waiting;
+  controller.abort(); release();
+  await result;
+  expect(transport).toHaveBeenCalledTimes(1);
+  expect(await store.readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  expect(await store.read()).toEqual(original);
+  expect(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement).toBeNull();
+});
+
+test.each(['identity_conflict', 'version_conflict'] as const)('M14 a cancelled late %s body cannot record a terminal result', async error => {
+  const store = await openItemStore(undefined, () => 'unit-late-conflict');
+  await store.replaceSnapshot([seeds[0]], 1700000000000);
+  await store.mutate({type: 'rename', id: seeds[0].id, title: 'Local unit', now: 1700000001000});
+  const pending = await store.readPending();
+  const controller = new AbortController();
+  let release!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  let entered!: () => void;
+  const waiting = new Promise<void>(resolve => {entered = resolve;});
+  const transport: JsonRequest = async () => ({status: 409, json: async () => {
+    entered(); await held;
+    return {error, item: {...seeds[0], title: 'Remote unit', version: 2, updatedAt: 1700000100000}, tombstone: null};
+  }});
+  const result = expect(new ForegroundSync(store, url, transport).synchronize(controller.signal))
+    .rejects.toThrow('Synchronization cancelled');
+  await waiting;
+  controller.abort(); release();
+  await result;
+  expect(await store.readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  expect(await store.readConflicts()).toEqual([]);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000000000);
+});
+
+test('M14 a cancelled late GET cannot replace the saved snapshot or successful-refresh time', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([seeds[0]], 1700000000000);
+  const controller = new AbortController();
+  let release!: () => void;
+  const held = new Promise<void>(resolve => {release = resolve;});
+  let entered!: () => void;
+  const waiting = new Promise<void>(resolve => {entered = resolve;});
+  const transport: JsonRequest = async (_address, options) => {
+    expect(options.method).toBe('GET');
+    expect(options.signal).toBe(controller.signal);
+    return {status: 200, json: async () => {
+      entered(); await held;
+      return {items: [{...seeds[0], title: 'Later remote', version: 2, updatedAt: 1700000100000}]};
+    }};
+  };
+  const sync = new ForegroundSync(store, url, transport);
+  const result = expect(sync.synchronize(controller.signal)).rejects.toThrow('Synchronization cancelled');
+  await waiting;
+  controller.abort(); release();
+  await result;
+  expect(await store.read()).toEqual([seeds[0]]);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000000000);
+  expect(sync.initialized).toBe(false);
+});
+
+test('M14 foreground abort closes one real request and reconnect replays the unchanged durable intent', async () => {
+  await request(`${url}/__m14-reset`, {method: 'POST', headers: {}});
+  const store = await openItemStore(undefined, () => 'unit-foreground-replay');
+  await store.mutate({type: 'create', title: 'Foreground unit', now: 1700000999000}, 'unit-foreground');
+  const original = await store.readPending();
+  const controller = new AbortController();
+  const sync = new ForegroundSync(store, url, request);
+  const result = expect(sync.uploadPending(controller.signal)).rejects.toThrow();
+  await waitPressure(state => state.requests[0]?.appliedAt != null && state.inFlightMutations === 1);
+  controller.abort();
+  await result;
+  const cancelled = await waitPressure(state => state.inFlightMutations === 0);
+  expect(cancelled.requests[0].finishedAt).toBeNull();
+  expect(cancelled.requests[0].disconnectedAt).not.toBeNull();
+  expect(await store.readPending()).toEqual(original.map(operation => ({...operation, dispatched: true})));
+  await sync.synchronize(new AbortController().signal);
+  expect(await store.readPending()).toEqual([]);
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 1});
+  const attempts = fixture.state().requests.filter(event => event.method === 'POST');
+  expect(attempts).toHaveLength(2);
+  expect(attempts[1].body).toEqual(attempts[0].body);
+  expect(await store.read()).toEqual([{id: 'unit-foreground-001', title: 'Foreground unit', completed: false,
+    version: 1, updatedAt: 1700001000000}]);
+});
+
 test('M10 background drain commits one canonical ACK without GET or refresh time, then acknowledges before releasing the runtime lock', async () => {
   const store = await openItemStore(undefined, () => 'unit-background-create');
   await store.replaceSnapshot([], 1700000100123);
diff --git a/android/app/build.gradle b/android/app/build.gradle
index a430f69..bc8f7d2 100644
--- a/android/app/build.gradle
+++ b/android/app/build.gradle
@@ -32,4 +32,5 @@ dependencies {
     androidTestImplementation("androidx.test:core:1.6.1")
     androidTestImplementation("androidx.test.ext:junit:1.2.1")
     androidTestImplementation("androidx.test.uiautomator:uiautomator:2.3.0")
+    androidTestImplementation 'com.github.requery:sqlite-android:3.49.0'
 }
diff --git a/android/app/src/androidTest/assets/m14-inputs.json b/android/app/src/androidTest/assets/m14-inputs.json
new file mode 100644
index 0000000..ed70412
--- /dev/null
+++ b/android/app/src/androidTest/assets/m14-inputs.json
@@ -0,0 +1,149 @@
+{
+  "profile": "phase-1",
+  "thread": "M14",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "568d620c2d25af44764b8b591d89bc32c0b786f8",
+  "identityPrefix": "pressure",
+  "items": [
+    {
+      "id": "pressure-001",
+      "title": "Pressure 001",
+      "completed": false
+    },
+    {
+      "id": "pressure-002",
+      "title": "Pressure 002",
+      "completed": false
+    },
+    {
+      "id": "pressure-003",
+      "title": "Pressure 003",
+      "completed": false
+    },
+    {
+      "id": "pressure-004",
+      "title": "Pressure 004",
+      "completed": false
+    },
+    {
+      "id": "pressure-005",
+      "title": "Pressure 005",
+      "completed": false
+    },
+    {
+      "id": "pressure-006",
+      "title": "Pressure 006",
+      "completed": false
+    },
+    {
+      "id": "pressure-007",
+      "title": "Pressure 007",
+      "completed": false
+    },
+    {
+      "id": "pressure-008",
+      "title": "Pressure 008",
+      "completed": false
+    },
+    {
+      "id": "pressure-009",
+      "title": "Pressure 009",
+      "completed": false
+    },
+    {
+      "id": "pressure-010",
+      "title": "Pressure 010",
+      "completed": false
+    },
+    {
+      "id": "pressure-011",
+      "title": "Pressure 011",
+      "completed": false
+    },
+    {
+      "id": "pressure-012",
+      "title": "Pressure 012",
+      "completed": false
+    }
+  ],
+  "envelopes": [
+    {
+      "itemId": "pressure-001",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-001\",\"title\":\"Pressure 001\"}}",
+      "payloadHash": "ae07b5d93d038ef1068755451d9cf6ed6fd415234fc834bb01e4b1698e2a713f"
+    },
+    {
+      "itemId": "pressure-002",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-002\",\"title\":\"Pressure 002\"}}",
+      "payloadHash": "29e613dfa77b8e6910680f063fa0c904cb972a580c83d6fce4ab353bcf19e9b5"
+    },
+    {
+      "itemId": "pressure-003",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-003\",\"title\":\"Pressure 003\"}}",
+      "payloadHash": "26b52518fe6f7145c3702c0eeb1c254888cc33cb0640bfd899ccd658d22cbdac"
+    },
+    {
+      "itemId": "pressure-004",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-004\",\"title\":\"Pressure 004\"}}",
+      "payloadHash": "6ae7172640b9c23da3c0ff578611b1243a1f6b9ad688bf45104be7db5340def5"
+    },
+    {
+      "itemId": "pressure-005",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-005\",\"title\":\"Pressure 005\"}}",
+      "payloadHash": "98c1b632b8da7811e6df23fc13e2bd685760d00bb84cbe74e067339952825285"
+    },
+    {
+      "itemId": "pressure-006",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-006\",\"title\":\"Pressure 006\"}}",
+      "payloadHash": "7b9ec807cd9363b8935959287284a4f18ae08dbeac85dc775351ec9d39793a25"
+    },
+    {
+      "itemId": "pressure-007",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-007\",\"title\":\"Pressure 007\"}}",
+      "payloadHash": "3013081e7f0959ad4c2b60f1aeab7df2110a594176f91d9f5eb0869e22b439d4"
+    },
+    {
+      "itemId": "pressure-008",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-008\",\"title\":\"Pressure 008\"}}",
+      "payloadHash": "675bf54598ee25e8f61be10926a97d607fa94752b8762bfab7a9759608bf7f9f"
+    },
+    {
+      "itemId": "pressure-009",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-009\",\"title\":\"Pressure 009\"}}",
+      "payloadHash": "d452b2dd73c5040e79d46ca57c1c869c5dd60204619931dba0d33bf4a40c13d0"
+    },
+    {
+      "itemId": "pressure-010",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-010\",\"title\":\"Pressure 010\"}}",
+      "payloadHash": "6be83218520a3faccb5e28ecb7503aec56bdfbc4dacc631686dc22665dba1861"
+    },
+    {
+      "itemId": "pressure-011",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-011\",\"title\":\"Pressure 011\"}}",
+      "payloadHash": "861951569c5960b3d77c6986561778b6398deb36d6afa5e8e138a153465bbd6c"
+    },
+    {
+      "itemId": "pressure-012",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-012\",\"title\":\"Pressure 012\"}}",
+      "payloadHash": "f85243c5292cc488ec16889c3fe25a4c58fcc51564e47ec21e0fedf120245948"
+    }
+  ],
+  "nativeScheduleRequests": 4,
+  "foregroundBurstRequests": 1,
+  "foregroundReconnectRequests": 1,
+  "barrierAcknowledgments": 3,
+  "responseDelayMs": 500,
+  "maxConcurrentMutations": 2,
+  "firstTimestamp": 1700001000000,
+  "finalNextTimestamp": 1700001012000,
+  "uiTimeoutMs": 15000,
+  "coordinatorTimeoutMs": 5000,
+  "networkTimeoutMs": 10000,
+  "settlementTimeoutMs": 10000,
+  "drainTimeoutMs": 30000,
+  "fixturePort": 18081,
+  "setup": "Existing React testIdentityPrefix prop only. All12 creates use production UI/SQLite; generated distinct mutation identities captured before dispatch. No queue or WorkDatabase injection.",
+  "barrier": "One consistent native SQLite read proves exactly3 durableACK before offline/cancellation; server receipt count alone is not accepted.",
+  "schedulerAttribution": "One preattached instrumentation session, actual nonforced null-namespace JobScheduler callback, Greedy enabled. No new process-death claim.",
+  "burstAccounting": "4 native schedule calls +1 accepted ordinary foreground Sync during active initial drain =5; reconnect foreground Sync is separate."
+}
diff --git a/android/app/src/androidTest/java/com/mse/reactnative/M14PressureTest.java b/android/app/src/androidTest/java/com/mse/reactnative/M14PressureTest.java
new file mode 100644
index 0000000..9902f0f
--- /dev/null
+++ b/android/app/src/androidTest/java/com/mse/reactnative/M14PressureTest.java
@@ -0,0 +1,1376 @@
+package com.mse.reactnative;
+
+import android.app.Activity;
+import android.app.Application;
+import android.app.Instrumentation;
+import android.content.Context;
+import android.database.Cursor;
+import android.database.sqlite.SQLiteDatabase;
+import android.graphics.Rect;
+import android.net.ConnectivityManager;
+import android.net.Network;
+import android.net.NetworkCapabilities;
+import android.net.NetworkInfo;
+import android.os.Build;
+import android.os.Bundle;
+import android.os.Handler;
+import android.os.Looper;
+import android.os.ParcelFileDescriptor;
+import android.os.Process;
+import android.os.SystemClock;
+import android.provider.Settings;
+import android.util.Base64;
+import android.util.Log;
+import android.view.View;
+import android.view.ViewGroup;
+import android.view.inputmethod.InputMethodManager;
+import android.widget.EditText;
+import android.widget.TextView;
+import androidx.test.core.app.ActivityScenario;
+import androidx.test.ext.junit.runners.AndroidJUnit4;
+import androidx.test.platform.app.InstrumentationRegistry;
+import androidx.test.uiautomator.By;
+import androidx.test.uiautomator.UiDevice;
+import androidx.test.uiautomator.UiObject2;
+import androidx.test.uiautomator.Until;
+import androidx.work.WorkInfo;
+import androidx.work.WorkManager;
+import com.facebook.react.ReactRootView;
+import com.facebook.react.bridge.NativeModule;
+import com.facebook.react.bridge.PromiseImpl;
+import com.facebook.react.bridge.ReactContext;
+import com.facebook.react.bridge.ReadableMap;
+import com.facebook.react.modules.network.NetworkingModule;
+import com.facebook.react.modules.network.CustomClientBuilder;
+import java.io.ByteArrayOutputStream;
+import java.io.File;
+import java.io.FileInputStream;
+import java.io.FileOutputStream;
+import java.io.InputStream;
+import java.io.IOException;
+import java.lang.reflect.Field;
+import java.net.HttpURLConnection;
+import java.net.InetSocketAddress;
+import java.net.Proxy;
+import java.net.URL;
+import java.nio.charset.StandardCharsets;
+import java.security.MessageDigest;
+import java.util.Arrays;
+import java.util.HashMap;
+import java.util.HashSet;
+import java.util.List;
+import java.util.Map;
+import java.util.Set;
+import java.util.concurrent.CountDownLatch;
+import java.util.concurrent.ExecutorService;
+import java.util.concurrent.Executors;
+import java.util.concurrent.Future;
+import java.util.concurrent.FutureTask;
+import java.util.concurrent.TimeUnit;
+import java.util.concurrent.atomic.AtomicBoolean;
+import java.util.concurrent.atomic.AtomicInteger;
+import java.util.concurrent.atomic.AtomicLong;
+import java.util.concurrent.atomic.AtomicReference;
+import java.util.regex.Pattern;
+import okhttp3.Call;
+import okhttp3.Connection;
+import okhttp3.EventListener;
+import okhttp3.OkHttpClient;
+import okhttp3.Protocol;
+import okhttp3.Request;
+import okhttp3.Response;
+import org.json.JSONArray;
+import org.json.JSONObject;
+import org.junit.Test;
+import org.junit.runner.RunWith;
+import static org.junit.Assert.*;
+
+/**
+ * One root-charged M14 invocation against the unchanged application.
+ * The external runner owns installation, fixture reset, initial offline state,
+ * final independent HTTP audit and cleanup. No Worker or JS implementation is replaced.
+ */
+@RunWith(AndroidJUnit4.class)
+public class M14PressureTest {
+    private static final String PACKAGE = "com.mse.reactnative";
+    private static final String UNIQUE_WORK = "item-uploads";
+    private static final String SERVICE = "androidx.work.impl.background.systemjob.SystemJobService";
+    private static final long OBSERVE_MS = 20; // Fixed local observation, not a scheduling policy.
+    private final Instrumentation instrumentation = InstrumentationRegistry.getInstrumentation();
+    private final Context context = instrumentation.getTargetContext();
+    private final MainApplication application = (MainApplication) context.getApplicationContext();
+    private final UiDevice device = UiDevice.getInstance(instrumentation);
+    private final File output = new File(context.getFilesDir(), "m14-evidence");
+    private final File itemsFile = new File(context.getFilesDir(), "items.db");
+    private final File workFile = new File(context.getNoBackupFilesDir(), "androidx.work.workdb");
+    private final Object evidenceLock = new Object();
+    private final JSONObject result = object("status", "RUNNING");
+    private final JSONArray events = new JSONArray();
+    private final JSONArray nativeRequests = new JSONArray();
+    private final JSONArray foregroundRequests = new JSONArray();
+    private final JSONObject checkpoints = new JSONObject();
+    private final AtomicInteger shellOrdinal = new AtomicInteger();
+    private final AtomicLong barrierAt = new AtomicLong();
+    private final AtomicReference<Throwable> watcherFailure = new AtomicReference<>();
+    private final AtomicBoolean stopWatcher = new AtomicBoolean();
+    private final CountDownLatch watcherReady = new CountDownLatch(1);
+    private final CountDownLatch barrierDone = new CountDownLatch(1);
+    private final ExecutorService shellExecutor = Executors.newFixedThreadPool(2, runnable -> {
+        Thread thread = new Thread(runnable, "M14-shell");
+        thread.setDaemon(true);
+        return thread;
+    });
+    private final int pid = Process.myPid();
+    private JSONObject input;
+    private MainActivity activity;
+    private ReactContext reactContext;
+    private OkHttpClient productionClient;
+    private JSONArray originalPending;
+    private String workId;
+    private String cycleId;
+    private Thread watcher;
+    private static final int REQUEST_TRACE_LIMIT = 384;
+    private final Object requestTraceLock = new Object();
+    private final JSONArray requestTrace = new JSONArray();
+    private int droppedRequestEvents;
+    private int requestObservationErrors;
+    private CustomClientBuilder previousClientBuilder;
+    private CustomClientBuilder observationClientBuilder;
+    private Field customClientBuilderField;
+    private boolean requestObservationInstalled;
+
+    private static JSONObject object(Object... pairs) {
+        JSONObject value = new JSONObject();
+        try {
+            for (int i = 0; i < pairs.length; i += 2) {
+                value.put((String) pairs[i], pairs[i + 1] == null ? JSONObject.NULL : pairs[i + 1]);
+            }
+        } catch (Exception error) { throw new IllegalStateException(error); }
+        return value;
+    }
+
+    private static byte[] bytes(InputStream stream) throws Exception {
+        ByteArrayOutputStream out = new ByteArrayOutputStream();
+        byte[] buffer = new byte[8192];
+        int count;
+        while ((count = stream.read(buffer)) != -1) out.write(buffer, 0, count);
+        return out.toByteArray();
+    }
+
+    private static String sha(byte[] data) throws Exception {
+        StringBuilder text = new StringBuilder();
+        for (byte value : MessageDigest.getInstance("SHA-256").digest(data)) {
+            text.append(String.format(java.util.Locale.ROOT, "%02x", value & 255));
+        }
+        return text.toString();
+    }
+
+    private void write(String name, byte[] data) throws Exception {
+        synchronized (evidenceLock) {
+            try (FileOutputStream stream = new FileOutputStream(new File(output, name))) {
+                stream.write(data);
+            }
+        }
+    }
+
+    private void writeJson(String name, Object value) throws Exception {
+        write(name, value.toString().getBytes(StandardCharsets.UTF_8));
+    }
+
+    private void save() throws Exception {
+        synchronized (evidenceLock) {
+            result.put("nativeScheduleRequests", nativeRequests);
+            result.put("foregroundRequests", foregroundRequests);
+            result.put("checkpoints", checkpoints);
+            writeJson("result.json", result);
+            writeJson("events.json", events);
+        }
+    }
+
+    private void put(String key, Object value) throws Exception {
+        synchronized (evidenceLock) { result.put(key, value); save(); }
+    }
+
+    private void event(String name, JSONObject detail) throws Exception {
+        JSONObject recorded;
+        synchronized (evidenceLock) {
+            recorded = new JSONObject(detail.toString());
+            recorded.put("event", name).put("at", System.currentTimeMillis())
+                    .put("elapsed", SystemClock.elapsedRealtime()).put("pid", Process.myPid());
+            events.put(recorded);
+            writeJson("events.json", events);
+        }
+        Log.i("M14Pressure", recorded.toString());
+    }
+
+    private long timeout(String name) throws Exception { return input.getLong(name); }
+
+    private interface Check { boolean ready() throws Exception; }
+
+    private void await(String label, long milliseconds, Check condition) throws Exception {
+        long deadline = SystemClock.elapsedRealtime() + milliseconds;
+        while (!condition.ready()) {
+            Throwable failed = watcherFailure.get();
+            if (failed != null) throw new AssertionError("ACK watcher failed", failed);
+            assertTrue("Timed out: " + label, SystemClock.elapsedRealtime() < deadline);
+            Thread.sleep(OBSERVE_MS);
+        }
+    }
+
+    private void onMain(Runnable action) throws Exception {
+        CountDownLatch done = new CountDownLatch(1);
+        AtomicReference<Throwable> failed = new AtomicReference<>();
+        new Handler(Looper.getMainLooper()).post(() -> {
+            try { action.run(); } catch (Throwable error) { failed.set(error); }
+            finally { done.countDown(); }
+        });
+        assertTrue("UI callback did not settle", done.await(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS));
+        if (failed.get() != null) throw new AssertionError("UI callback failed", failed.get());
+    }
+
+    private static View described(View view, String description) {
+        if (description.contentEquals(view.getContentDescription() == null ? "" : view.getContentDescription())) return view;
+        if (view instanceof ViewGroup) {
+            ViewGroup group = (ViewGroup) view;
+            for (int i = 0; i < group.getChildCount(); i++) {
+                View found = described(group.getChildAt(i), description);
+                if (found != null) return found;
+            }
+        }
+        return null;
+    }
+
+    private boolean hasDescription(String description) throws Exception {
+        AtomicBoolean found = new AtomicBoolean();
+        onMain(() -> found.set(described(activity.getWindow().getDecorView(), description) != null));
+        return found.get();
+    }
+
+    private UiObject2 control(String description) throws Exception {
+        UiObject2 found = device.wait(Until.findObject(By.desc(description)), timeout("uiTimeoutMs"));
+        assertNotNull("Missing control: " + description, found);
+        return found;
+    }
+
+    private void ready() throws Exception {
+        await("Local storage ready", timeout("uiTimeoutMs"), () -> hasDescription("Local storage ready"));
+    }
+
+    private void captureUi(String label) throws Exception {
+        device.dumpWindowHierarchy(new File(output, label + ".xml"));
+        assertTrue("Screenshot failed: " + label, device.takeScreenshot(new File(output, label + ".png")));
+    }
+
+    private static String boundedText(Object value) {
+        String text = String.valueOf(value);
+        return text.length() <= 4096 ? text : text.substring(0, 4096) + " [truncated]";
+    }
+
+    private JSONObject diagnosticNetwork() throws Exception {
+        ConnectivityManager manager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
+        Network active = manager.getActiveNetwork();
+        return object("state", networkState(), "defaultNetwork", active == null ? null : active.toString(),
+                "defaultNetworkHandle", active == null ? null : active.getNetworkHandle(),
+                "capabilities", active == null ? null : boundedText(manager.getNetworkCapabilities(active)),
+                "linkProperties", active == null ? null : boundedText(manager.getLinkProperties(active)),
+                "socketBindingClaim", false, "at", System.currentTimeMillis(),
+                "elapsed", SystemClock.elapsedRealtime());
+    }
+
+    private void diagnosticViews(View view, int parent, JSONArray nodes) {
+        if (nodes.length() >= 512) return;
+        int index = nodes.length();
+        nodes.put(object("index", index, "parent", parent, "class", view.getClass().getName(),
+                "id", view.getId(), "description", boundedText(view.getContentDescription()),
+                "text", view instanceof TextView ? boundedText(((TextView) view).getText()) : null,
+                "visibility", view.getVisibility(), "enabled", view.isEnabled(),
+                "shown", view.isShown()));
+        if (view instanceof ViewGroup) {
+            ViewGroup group = (ViewGroup) view;
+            for (int i = 0; i < group.getChildCount() && nodes.length() < 512; i++) {
+                diagnosticViews(group.getChildAt(i), index, nodes);
+            }
+        }
+    }
+
+    private interface DiagnosticRead { Object read() throws Exception; }
+
+    private void failureStep(JSONObject detail, String name, DiagnosticRead read) {
+        try { detail.put(name, read.read()); }
+        catch (Throwable observation) {
+            if (observation instanceof InterruptedException) Thread.currentThread().interrupt();
+            try { detail.put(name + "ObservationError", boundedText(Log.getStackTraceString(observation))); }
+            catch (Throwable ignored) { /* Never replace the scenario failure. */ }
+        }
+        try { writeJson("reconnect-failure.json", detail); }
+        catch (Throwable ignored) { /* A later summary records whether capture completed. */ }
+    }
+
+    private void captureReconnectFailure(Throwable original) {
+        // The original await has already failed. No extra drain/retry or action
+        // runs here. One bounded, daemon, read-only collector precedes teardown.
+        FutureTask<Void> capture = new FutureTask<>(() -> {
+            JSONObject detail = object("originalError", boundedText(original),
+                    "startedAt", System.currentTimeMillis(), "startedElapsed", SystemClock.elapsedRealtime());
+            failureStep(detail, "dispatcher", this::dispatcher);
+            failureStep(detail, "network", this::diagnosticNetwork);
+            failureStep(detail, "requests", this::requestObservation);
+            failureStep(detail, "ui", () -> {
+                AtomicReference<JSONObject> observed = new AtomicReference<>();
+                onMain(() -> {
+                    JSONArray nodes = new JSONArray();
+                    diagnosticViews(activity.getWindow().getDecorView(), -1, nodes);
+                    observed.set(object("nodes", nodes, "nodeLimit", 512,
+                            "limitReached", nodes.length() >= 512, "onUiThread", true,
+                            "activityIdentity", System.identityHashCode(activity),
+                            "finishing", activity.isFinishing(), "destroyed", activity.isDestroyed()));
+                });
+                return observed.get();
+            });
+            // Query the same read-only engines without raw live-file copies:
+            // closing an arbitrary file descriptor could release SQLite locks.
+            // Root may copy bytes externally after stopping the failed app.
+            if (!Thread.currentThread().isInterrupted()) failureStep(detail, "items", () -> database("reconnect-failure-items", itemsFile, false));
+            if (!Thread.currentThread().isInterrupted()) failureStep(detail, "work", () -> database("reconnect-failure-work", workFile, false));
+            if (!Thread.currentThread().isInterrupted()) failureStep(detail, "preferences", () -> preferences("reconnect-failure"));
+            failureStep(detail, "completedElapsed", SystemClock::elapsedRealtime);
+            return null;
+        });
+        Thread collector = new Thread(capture, "M14-failure-observation");
+        collector.setDaemon(true);
+        try {
+            collector.start();
+            capture.get(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS);
+            put("reconnectFailureObservation", object("status", "COMPLETED", "file", "reconnect-failure.json",
+                    "captureBudgetMs", timeout("coordinatorTimeoutMs")));
+        } catch (Throwable observation) {
+            capture.cancel(true);
+            try {
+                put("reconnectFailureObservation", object("status", "INCOMPLETE", "file", "reconnect-failure.json",
+                        "error", boundedText(Log.getStackTraceString(observation)),
+                        "captureBudgetMs", timeout("coordinatorTimeoutMs"),
+                        "readOnlyCollectorAlive", collector.isAlive()));
+            } catch (Throwable ignored) { /* Preserve original failure even if evidence IO fails. */ }
+        }
+    }
+
+    private static String commandLine(String... argv) {
+        if (argv == null || argv.length == 0) throw new IllegalArgumentException("Empty command");
+        for (String argument : argv) {
+            if (argument == null || !argument.matches("[A-Za-z0-9._/-]+")) {
+                throw new IllegalArgumentException("Unsupported fixed command token: " + argument);
+            }
+        }
+        // UiAutomationConnection uses Runtime.exec(String), not a shell parser.
+        return String.join(" ", argv);
+    }
+
+    private static void closeDescriptors(ParcelFileDescriptor[] descriptors) {
+        if (descriptors == null) return;
+        for (ParcelFileDescriptor descriptor : descriptors) {
+            if (descriptor != null) {
+                try { descriptor.close(); } catch (Exception ignored) { /* Preserve the original failure. */ }
+            }
+        }
+    }
+
+    private byte[] shellBytes(ParcelFileDescriptor descriptor, String file) throws Exception {
+        ByteArrayOutputStream captured = new ByteArrayOutputStream();
+        try (InputStream stream = new ParcelFileDescriptor.AutoCloseInputStream(descriptor);
+             FileOutputStream raw = new FileOutputStream(new File(output, file))) {
+            byte[] buffer = new byte[8192];
+            int count;
+            while ((count = stream.read(buffer)) != -1) {
+                raw.write(buffer, 0, count); // Retain partial output if the bounded read fails.
+                captured.write(buffer, 0, count);
+            }
+        }
+        return captured.toByteArray();
+    }
+
+    private String shell(String label, String... argv) throws Exception {
+        int number = shellOrdinal.getAndIncrement();
+        String file = String.format(java.util.Locale.ROOT, "shell-%03d-%s.txt", number, label);
+        String stderrFile = String.format(java.util.Locale.ROOT, "shell-%03d-%s.stderr.txt", number, label);
+        String command = commandLine(argv);
+        JSONObject entry = object("label", label, "argv", new JSONArray(Arrays.asList(argv)),
+                "command", command, "startedAt", System.currentTimeMillis(),
+                "startedElapsed", SystemClock.elapsedRealtime(), "outputFile", file,
+                "stderrFile", stderrFile, "exitStatusAvailable", false);
+        AtomicReference<ParcelFileDescriptor[]> descriptors = new AtomicReference<>();
+        AtomicBoolean cancelled = new AtomicBoolean();
+        Future<String> running = shellExecutor.submit(() -> {
+            ParcelFileDescriptor[] fds = instrumentation.getUiAutomation().executeShellCommandRwe(command);
+            descriptors.set(fds);
+            FutureTask<byte[]> stderr = null;
+            try {
+                // Also close descriptors returned after the caller's timeout.
+                if (cancelled.get()) throw new IllegalStateException("Command acquisition exceeded its timeout");
+                assertNotNull("Shell did not return descriptors", fds);
+                assertEquals(3, fds.length);
+                for (ParcelFileDescriptor fd : fds) assertNotNull(fd);
+                fds[1].close(); // These fixed commands have no stdin.
+                stderr = new FutureTask<>(() -> shellBytes(fds[2], stderrFile));
+                // Do not occupy the shared two-thread executor waiting for its
+                // own reader tasks when watcher and main commands overlap.
+                Thread reader = new Thread(stderr, "M14-shell-stderr-" + number);
+                reader.setDaemon(true);
+                reader.start();
+                byte[] stdout = shellBytes(fds[0], file);
+                stderr.get();
+                return new String(stdout, StandardCharsets.UTF_8);
+            } finally {
+                closeDescriptors(fds);
+                if (stderr != null) stderr.cancel(true);
+            }
+        });
+        try {
+            String text = running.get(timeout("networkTimeoutMs"), TimeUnit.MILLISECONDS);
+            entry.put("pipesReadToEof", true); // EOF is not an OS exit-status assertion.
+            return text;
+        } catch (Exception error) {
+            entry.put("error", error.toString()).put("outcome", "failed_or_indeterminate");
+            cancelled.set(true);
+            closeDescriptors(descriptors.get());
+            running.cancel(true);
+            throw error;
+        } finally {
+            entry.put("endedAt", System.currentTimeMillis()).put("endedElapsed", SystemClock.elapsedRealtime());
+            event("shell", entry);
+        }
+    }
+
+    @SuppressWarnings("deprecation")
+    private JSONObject networkState() throws Exception {
+        ConnectivityManager manager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
+        NetworkInfo active = manager.getActiveNetworkInfo();
+        NetworkCapabilities caps = manager.getNetworkCapabilities(manager.getActiveNetwork());
+        return object("airplane_mode_on", Settings.Global.getString(context.getContentResolver(), "airplane_mode_on"),
+                "wifi_on", Settings.Global.getString(context.getContentResolver(), "wifi_on"),
+                "mobile_data", Settings.Global.getString(context.getContentResolver(), "mobile_data"),
+                "connected", active != null && active.isConnected(),
+                "validated", caps != null && caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED));
+    }
+
+    private boolean networkMatches(JSONObject state, boolean online) throws Exception {
+        return state.getString("airplane_mode_on").equals(online ? "0" : "1")
+                && state.getString("wifi_on").equals(online ? "1" : "0")
+                && state.getString("mobile_data").equals(online ? "1" : "0")
+                && state.getBoolean("connected") == online
+                && (!online || state.getBoolean("validated"));
+    }
+
+    private void requestNetwork(String label, boolean online, boolean initial) throws Exception {
+        event("network-request", object("label", label, "online", online));
+        shell(label + "-airplane", "cmd", "connectivity", "airplane-mode", online ? "disable" : "enable");
+        if (initial) assertBeforeBarrier("initial connectivity transition");
+        shell(label + "-wifi", "svc", "wifi", online ? "enable" : "disable");
+        if (initial) assertBeforeBarrier("initial connectivity transition");
+        shell(label + "-data", "svc", "data", online ? "enable" : "disable");
+        if (initial) assertBeforeBarrier("initial connectivity transition");
+    }
+
+    private void observeNetwork(String label, boolean online) throws Exception {
+        await(label + " connectivity", timeout("networkTimeoutMs"), () -> networkMatches(networkState(), online));
+        event("network-observed", object("label", label, "state", networkState()));
+    }
+
+    private void changeNetwork(String label, boolean online, boolean initial) throws Exception {
+        requestNetwork(label, online, initial);
+        observeNetwork(label, online);
+    }
+
+    private JSONObject fixtureState(String label) throws Exception {
+        // Called only while online. Offline checkpoints use native observations.
+        String endpoint = "http://10.0.2.2:" + input.getInt("fixturePort") + "/__m14-state";
+        JSONObject timing = object("label", label, "url", endpoint, "startedAt", System.currentTimeMillis(),
+                "startedElapsed", SystemClock.elapsedRealtime());
+        HttpURLConnection connection = (HttpURLConnection) new URL(endpoint).openConnection();
+        connection.setConnectTimeout((int) timeout("coordinatorTimeoutMs"));
+        connection.setReadTimeout((int) timeout("coordinatorTimeoutMs"));
+        try {
+            int status = connection.getResponseCode();
+            byte[] raw;
+            try (InputStream stream = status >= 400 ? connection.getErrorStream() : connection.getInputStream()) {
+                raw = bytes(stream);
+            }
+            String file = "control-" + label + ".json";
+            write(file, raw);
+            timing.put("status", status).put("outputFile", file);
+            assertEquals("Fixture control status", 200, status);
+            return new JSONObject(new String(raw, StandardCharsets.UTF_8));
+        } finally {
+            connection.disconnect();
+            timing.put("endedAt", System.currentTimeMillis()).put("endedElapsed", SystemClock.elapsedRealtime());
+            event("fixture-control", timing);
+        }
+    }
+
+    private static JSONObject cursorRow(Cursor cursor) throws Exception {
+        JSONObject row = new JSONObject();
+        for (int i = 0; i < cursor.getColumnCount(); i++) {
+            Object cell;
+            switch (cursor.getType(i)) {
+                case Cursor.FIELD_TYPE_NULL: cell = JSONObject.NULL; break;
+                case Cursor.FIELD_TYPE_INTEGER: cell = cursor.getLong(i); break;
+                case Cursor.FIELD_TYPE_FLOAT: cell = cursor.getDouble(i); break;
+                case Cursor.FIELD_TYPE_BLOB: cell = object("base64", Base64.encodeToString(cursor.getBlob(i), Base64.NO_WRAP)); break;
+                default: cell = cursor.getString(i);
+            }
+            row.put(cursor.getColumnName(i), cell);
+        }
+        return row;
+    }
+
+    private io.requery.android.database.sqlite.SQLiteDatabase openItemObserver() throws Exception {
+        // Use the production SQLite engine, but never its writer connection.
+        io.requery.android.database.sqlite.SQLiteDatabase db =
+                io.requery.android.database.sqlite.SQLiteDatabase.openDatabase(itemsFile.getPath(), null,
+                        io.requery.android.database.sqlite.SQLiteDatabase.OPEN_READONLY,
+                        damaged -> { throw new AssertionError("Item observer refuses corruption recovery or deletion"); });
+        boolean returned = false;
+        try {
+            assertTrue("Item observer must remain read-only", db.isReadOnly());
+            event("item-observer-open", object("sourcePath", itemsFile.getPath(),
+                    "engineClass", db.getClass().getName(), "readOnly", db.isReadOnly(),
+                    "connectionIdentity", System.identityHashCode(db), "writerHandleBorrowed", false));
+            returned = true;
+            return db;
+        } finally {
+            if (!returned) db.close();
+        }
+    }
+
+    private JSONObject database(String label, File source) throws Exception {
+        return database(label, source, true);
+    }
+
+    private JSONObject database(String label, File source, boolean copyRawFiles) throws Exception {
+        long began = SystemClock.elapsedRealtime();
+        JSONObject tables = new JSONObject();
+        JSONArray schema = new JSONArray();
+        int version;
+        String engine;
+        boolean itemSource = source.equals(itemsFile);
+        try (io.requery.android.database.sqlite.SQLiteDatabase item = itemSource ? openItemObserver() : null;
+             SQLiteDatabase work = itemSource ? null
+                     : SQLiteDatabase.openDatabase(source.getPath(), null, SQLiteDatabase.OPEN_READONLY)) {
+            version = itemSource ? item.getVersion() : work.getVersion();
+            engine = itemSource ? item.getClass().getName() : work.getClass().getName();
+            String schemaSql = "SELECT name,sql FROM sqlite_master WHERE type='table' ORDER BY name";
+            try (Cursor cursor = itemSource ? item.rawQuery(schemaSql, null) : work.rawQuery(schemaSql, null)) {
+                while (cursor.moveToNext()) schema.put(cursorRow(cursor));
+            }
+            for (int i = 0; i < schema.length(); i++) {
+                String name = schema.getJSONObject(i).getString("name");
+                assertTrue("Unexpected native table name", name.matches("[A-Za-z_][A-Za-z_0-9]*"));
+                JSONArray rows = new JSONArray();
+                String rowsSql = "SELECT * FROM \"" + name + "\" ORDER BY rowid";
+                try (Cursor cursor = itemSource ? item.rawQuery(rowsSql, null) : work.rawQuery(rowsSql, null)) {
+                    while (cursor.moveToNext()) rows.put(cursorRow(cursor));
+                }
+                tables.put(name, rows);
+            }
+        }
+        JSONObject snapshot = object("sourcePath", source.getPath(), "schemaVersion", version,
+                "observerEngine", engine, "schema", schema, "tables", tables, "queryStartedElapsed", began,
+                "queryEndedElapsed", SystemClock.elapsedRealtime(), "at", System.currentTimeMillis());
+        JSONObject files = new JSONObject();
+        JSONObject hashes = new JSONObject();
+        // Preserve source bytes separately; do not checkpoint or rewrite either database.
+        // The exact barrier is a separate one-statement JSON observation, not this later copy.
+        for (String suffix : new String[]{"", "-wal", "-shm"}) {
+            if (!copyRawFiles) break;
+            File current = new File(source.getPath() + suffix);
+            if (!current.exists()) continue;
+            byte[] raw;
+            try (InputStream stream = new FileInputStream(current)) { raw = bytes(stream); }
+            String name = label + ".db" + suffix;
+            write(name, raw);
+            String digest = sha(raw);
+            hashes.put(name, digest);
+            files.put(name, object("sha256", digest, "bytes", raw.length,
+                    "sourcePath", current.getPath(), "copiedAt", System.currentTimeMillis(),
+                    "copiedElapsed", SystemClock.elapsedRealtime()));
+        }
+        snapshot.put("nativeFiles", files).put("jsonFile", label + ".json")
+                .put("baseFile", copyRawFiles ? label + ".db" : JSONObject.NULL).put("rawSha256", hashes);
+        if (!copyRawFiles) snapshot.put("rawCopyOmitted", "Live failure observation has no quiescence guarantee; queried JSON only");
+        writeJson(label + ".json", snapshot);
+        return snapshot;
+    }
+
+    private JSONObject preferences(String label) throws Exception {
+        File source = new File(context.getApplicationInfo().dataDir, "shared_prefs/item-background-sync.xml");
+        byte[] raw;
+        try (InputStream stream = new FileInputStream(source)) { raw = bytes(stream); }
+        write(label + "-cycle.xml", raw);
+        // getAll is evidence alongside, never an editor/commit from the test.
+        return object("file", label + "-cycle.xml", "sha256", sha(raw),
+                "values", new JSONObject(context.getSharedPreferences("item-background-sync", Context.MODE_PRIVATE).getAll()));
+    }
+
+    private JSONArray uniqueWork() throws Exception {
+        List<WorkInfo> work = WorkManager.getInstance(context).getWorkInfosForUniqueWork(UNIQUE_WORK)
+                .get(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS);
+        JSONArray values = new JSONArray();
+        for (WorkInfo info : work) values.put(object("id", info.getId().toString(),
+                "state", info.getState().name(), "runAttemptCount", info.getRunAttemptCount()));
+        return values;
+    }
+
+    private JSONObject workObservation() throws Exception {
+        try (SQLiteDatabase db = SQLiteDatabase.openDatabase(workFile.getPath(), null, SQLiteDatabase.OPEN_READONLY);
+             Cursor cursor = db.rawQuery("SELECT w.id,w.state,w.run_attempt_count,w.generation,w.schedule_requested_at,"
+                     + "(SELECT COUNT(*) FROM WorkSpec) AS work_count "
+                     + "FROM WorkSpec w JOIN WorkName n ON w.id=n.work_spec_id WHERE n.name=?",
+                     new String[]{UNIQUE_WORK})) {
+            assertTrue("Missing production unique work", cursor.moveToFirst());
+            JSONObject value = cursorRow(cursor);
+            assertFalse("Duplicate active initial work", cursor.moveToNext());
+            return value;
+        }
+    }
+
+    private JSONObject dispatcher() {
+        JSONArray running = new JSONArray();
+        JSONArray queued = new JSONArray();
+        for (Call call : productionClient.dispatcher().runningCalls()) {
+            running.put(object("method", call.request().method(), "url", call.request().url().toString(),
+                    "cancelled", call.isCanceled()));
+        }
+        for (Call call : productionClient.dispatcher().queuedCalls()) {
+            queued.put(object("method", call.request().method(), "url", call.request().url().toString(),
+                    "cancelled", call.isCanceled()));
+        }
+        return object("runningCount", running.length(), "queuedCount", queued.length(),
+                "running", running, "queued", queued, "at", System.currentTimeMillis(),
+                "elapsedRealtime", SystemClock.elapsedRealtime());
+    }
+
+    private void requestEvent(String phase, Call call, Object... detail) {
+        try {
+            JSONObject row = object("phase", phase, "at", System.currentTimeMillis(),
+                    "elapsed", SystemClock.elapsedRealtime(), "pid", pid);
+            if (call != null) {
+                row.put("callIdentity", System.identityHashCode(call)).put("tag", call.request().tag())
+                        .put("method", call.request().method()).put("url", call.request().url().toString())
+                        .put("cancelled", call.isCanceled());
+            }
+            for (int i = 0; i < detail.length; i += 2) {
+                Object value = detail[i + 1];
+                if (value instanceof Throwable) {
+                    Throwable error = (Throwable) value;
+                    value = object("class", error.getClass().getName(), "message", boundedText(error.getMessage()),
+                            "stack", boundedText(Log.getStackTraceString(error)));
+                } else if (value instanceof Connection) {
+                    Connection connection = (Connection) value;
+                    value = object("identity", System.identityHashCode(connection),
+                            "routeSocketAddress", String.valueOf(connection.route().socketAddress()),
+                            "proxy", String.valueOf(connection.route().proxy()),
+                            "protocol", String.valueOf(connection.protocol()),
+                            "socketIdentity", System.identityHashCode(connection.socket()),
+                            "localAddress", String.valueOf(connection.socket().getLocalSocketAddress()),
+                            "remoteAddress", String.valueOf(connection.socket().getRemoteSocketAddress()),
+                            "closed", connection.socket().isClosed());
+                } else if (value instanceof Response) {
+                    value = object("status", ((Response) value).code());
+                } else if (value instanceof InetSocketAddress || value instanceof Proxy || value instanceof Protocol) {
+                    value = boundedText(value);
+                }
+                row.put((String) detail[i], value == null ? JSONObject.NULL : value);
+            }
+            if (phase.equals("callStart") || phase.equals("callFailed")) {
+                try {
+                    ConnectivityManager manager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
+                    Network active = manager.getActiveNetwork();
+                    row.put("defaultNetworkAtObservation", active == null ? JSONObject.NULL : active.toString())
+                            .put("defaultNetworkHandle", active == null ? JSONObject.NULL : active.getNetworkHandle())
+                            .put("socketBindingClaim", false);
+                } catch (Throwable observation) {
+                    row.put("networkObservationError", boundedText(observation));
+                }
+            }
+            synchronized (requestTraceLock) {
+                if (requestTrace.length() < REQUEST_TRACE_LIMIT) requestTrace.put(row);
+                else droppedRequestEvents++;
+            }
+        } catch (Throwable observation) {
+            synchronized (requestTraceLock) { requestObservationErrors++; }
+        }
+    }
+
+    private JSONObject requestObservation() throws Exception {
+        synchronized (requestTraceLock) {
+            return object("events", new JSONArray(requestTrace.toString()), "eventLimit", REQUEST_TRACE_LIMIT,
+                    "dropped", droppedRequestEvents, "observationErrors", requestObservationErrors,
+                    "defaultNetworkIsNotSocketBinding", true, "perEventDatabaseOrFileIo", false);
+        }
+    }
+
+    private EventListener requestListener(Call call, EventListener original) {
+        // Prove NONE on each actual call, not by constructing a synthetic call.
+        // An unexpected existing listener is returned intact, preserving every
+        // callback (including ones not observed below); no partial delegation.
+        requestEvent("listenerSelected", call, "originalListener", original.getClass().getName(),
+                "originalIdentity", System.identityHashCode(original), "originalIsNone", original == EventListener.NONE,
+                "passiveObservationAvailable", original == EventListener.NONE);
+        if (original != EventListener.NONE) return original;
+        return new EventListener() {
+            @Override public void callStart(Call value) { requestEvent("callStart", value); }
+            @Override public void connectStart(Call value, InetSocketAddress address, Proxy proxy) {
+                requestEvent("connectStart", value, "address", address, "proxy", proxy);
+            }
+            @Override public void connectEnd(Call value, InetSocketAddress address, Proxy proxy, Protocol protocol) {
+                requestEvent("connectEnd", value, "address", address, "proxy", proxy, "protocol", protocol);
+            }
+            @Override public void connectFailed(Call value, InetSocketAddress address, Proxy proxy, Protocol protocol, IOException error) {
+                requestEvent("connectFailed", value, "address", address, "proxy", proxy, "protocol", protocol, "error", error);
+            }
+            @Override public void connectionAcquired(Call value, Connection connection) { requestEvent("connectionAcquired", value, "connection", connection); }
+            @Override public void connectionReleased(Call value, Connection connection) { requestEvent("connectionReleased", value, "connection", connection); }
+            @Override public void requestHeadersStart(Call value) { requestEvent("requestHeadersStart", value); }
+            @Override public void requestHeadersEnd(Call value, Request request) { requestEvent("requestHeadersEnd", value); }
+            @Override public void requestBodyEnd(Call value, long count) { requestEvent("requestBodyEnd", value, "byteCount", count); }
+            @Override public void requestFailed(Call value, IOException error) { requestEvent("requestFailed", value, "error", error); }
+            @Override public void responseHeadersStart(Call value) { requestEvent("responseHeadersStart", value); }
+            @Override public void responseHeadersEnd(Call value, Response response) { requestEvent("responseHeadersEnd", value, "response", response); }
+            @Override public void responseFailed(Call value, IOException error) { requestEvent("responseFailed", value, "error", error); }
+            @Override public void callEnd(Call value) { requestEvent("callEnd", value); }
+            @Override public void callFailed(Call value, IOException error) { requestEvent("callFailed", value, "error", error); }
+        };
+    }
+
+    private JSONObject clientPolicy(OkHttpClient.Builder builder) {
+        // Pinned4.9.2 getters: no new client, call, socket, or request is built.
+        return object("dispatcher", System.identityHashCode(builder.getDispatcher$okhttp()),
+                "pool", System.identityHashCode(builder.getConnectionPool$okhttp()),
+                "dns", System.identityHashCode(builder.getDns$okhttp()),
+                "proxy", System.identityHashCode(builder.getProxy$okhttp()),
+                "proxySelector", System.identityHashCode(builder.getProxySelector$okhttp()),
+                "socketFactory", System.identityHashCode(builder.getSocketFactory$okhttp()),
+                "sslSocketFactory", System.identityHashCode(builder.getSslSocketFactoryOrNull$okhttp()),
+                "cookieJar", System.identityHashCode(builder.getCookieJar$okhttp()),
+                "cache", System.identityHashCode(builder.getCache$okhttp()),
+                "authenticator", System.identityHashCode(builder.getAuthenticator$okhttp()),
+                "proxyAuthenticator", System.identityHashCode(builder.getProxyAuthenticator$okhttp()),
+                "hostnameVerifier", System.identityHashCode(builder.getHostnameVerifier$okhttp()),
+                "certificatePinner", System.identityHashCode(builder.getCertificatePinner$okhttp()),
+                "connectionSpecs", System.identityHashCode(builder.getConnectionSpecs$okhttp()),
+                "protocols", System.identityHashCode(builder.getProtocols$okhttp()),
+                "interceptors", System.identityHashCode(builder.interceptors()),
+                "networkInterceptors", System.identityHashCode(builder.networkInterceptors()),
+                "retryOnConnectionFailure", builder.getRetryOnConnectionFailure$okhttp(),
+                "followRedirects", builder.getFollowRedirects$okhttp(),
+                "followSslRedirects", builder.getFollowSslRedirects$okhttp(),
+                "callTimeoutMs", builder.getCallTimeout$okhttp(),
+                "connectTimeoutMs", builder.getConnectTimeout$okhttp(),
+                "readTimeoutMs", builder.getReadTimeout$okhttp(),
+                "writeTimeoutMs", builder.getWriteTimeout$okhttp(),
+                "pingIntervalMs", builder.getPingInterval$okhttp());
+    }
+
+    private void clientObservation(boolean install) throws Exception {
+        CountDownLatch done = new CountDownLatch(1);
+        AtomicReference<Throwable> failed = new AtomicReference<>();
+        // RN's static seam is not volatile. Install/restore on its owning native
+        // queue, with the existing bounded coordinator wait before proceeding.
+        reactContext.runOnNativeModulesQueueThread(() -> {
+            try {
+                if (install) {
+                    customClientBuilderField = NetworkingModule.class.getDeclaredField("customClientBuilder");
+                    customClientBuilderField.setAccessible(true);
+                    previousClientBuilder = (CustomClientBuilder) customClientBuilderField.get(null);
+                    observationClientBuilder = builder -> {
+                        if (previousClientBuilder != null) previousClientBuilder.apply(builder);
+                        // Preserve prior hook exceptions and every client policy.
+                        // This pinned public getter avoids building another client.
+                        try {
+                            EventListener.Factory originalFactory = builder.getEventListenerFactory$okhttp();
+                            JSONObject before = clientPolicy(builder);
+                            builder.eventListenerFactory(call -> {
+                                EventListener original = originalFactory.create(call);
+                                try { return requestListener(call, original); }
+                                catch (Throwable observation) {
+                                    requestEvent("listenerObservationError", call, "error", observation);
+                                    return original;
+                                }
+                            });
+                            JSONObject after = clientPolicy(builder);
+                            requestEvent("builderPolicy", null, "beforeObserver", before,
+                                    "afterObserver", after, "unchanged", before.toString().equals(after.toString()),
+                                    "originalFactory", originalFactory.getClass().getName());
+                        } catch (Throwable observation) {
+                            requestEvent("builderObservationError", null, "error", observation);
+                        }
+                    };
+                    NetworkingModule.setCustomClientBuilder(observationClientBuilder);
+                    requestObservationInstalled = true;
+                    requestEvent("seamInstalled", null, "previousIdentity", System.identityHashCode(previousClientBuilder),
+                            "installedIdentity", System.identityHashCode(observationClientBuilder),
+                            "dispatcherIdentity", System.identityHashCode(productionClient.dispatcher()),
+                            "poolIdentity", System.identityHashCode(productionClient.connectionPool()));
+                } else if (requestObservationInstalled) {
+                    Object before = customClientBuilderField.get(null);
+                    NetworkingModule.setCustomClientBuilder(previousClientBuilder);
+                    requestObservationInstalled = false;
+                    requestEvent("seamRestored", null, "ownedAtRestore", before == observationClientBuilder,
+                            "restoredExactPrevious", customClientBuilderField.get(null) == previousClientBuilder,
+                            "previousIdentity", System.identityHashCode(previousClientBuilder));
+                }
+            } catch (Throwable error) { failed.set(error); }
+            finally { done.countDown(); }
+        });
+        assertTrue("Passive client observation queue did not settle", done.await(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS));
+        if (failed.get() != null) throw new AssertionError("Passive client observation failed", failed.get());
+    }
+
+    private void sameRuntime(String label) throws Exception {
+        onMain(() -> {
+            assertEquals("Process changed", pid, Process.myPid());
+            assertSame(application, activity.getApplication());
+            assertSame(reactContext, application.getReactNativeHost().getReactInstanceManager().getCurrentReactContext());
+            assertFalse("Activity destroyed", activity.isDestroyed());
+        });
+        event("runtime", object("label", label, "applicationIdentity", System.identityHashCode(application),
+                "activityIdentity", System.identityHashCode(activity), "reactContextIdentity", System.identityHashCode(reactContext)));
+    }
+
+    private JSONObject checkpoint(String label, boolean ui) throws Exception {
+        sameRuntime(label);
+        JSONObject value = object("items", database(label + "-items", itemsFile),
+                "work", database(label + "-work", workFile), "preferences", preferences(label),
+                "uniqueWork", uniqueWork(), "dispatcher", dispatcher(), "network", networkState());
+        if (ui) {
+            captureUi(label);
+            value.put("uiXml", label + ".xml").put("uiPng", label + ".png");
+        }
+        synchronized (evidenceLock) { checkpoints.put(label, value); save(); }
+        return value;
+    }
+
+    private JSONObject ackObservation(io.requery.android.database.sqlite.SQLiteDatabase db) throws Exception {
+        // ONE SQLite statement supplies the commit-consistent barrier. No open
+        // cursor/transaction is retained while requesting offline or cancelling work.
+        String sql = "SELECT (SELECT COUNT(*) FROM pending_mutations) AS pendingCount,last_acknowledgement,"
+                + "(SELECT item_id FROM pending_mutations ORDER BY sequence LIMIT 1) AS firstRemaining,"
+                + "(SELECT COUNT(*) FROM pending_mutations WHERE item_id IN (?,?,?)) AS firstThreePending "
+                + "FROM sync_metadata WHERE singleton=1";
+        JSONObject row;
+        try (Cursor cursor = db.rawQuery(sql, new String[]{"pressure-001", "pressure-002", "pressure-003"})) {
+            assertTrue("Missing sync metadata", cursor.moveToFirst());
+            row = cursorRow(cursor);
+        }
+        row.put("observedAt", System.currentTimeMillis()).put("elapsedRealtime", SystemClock.elapsedRealtime());
+        if (!row.isNull("last_acknowledgement")) {
+            row.put("lastAcknowledgement", new JSONObject(row.getString("last_acknowledgement")));
+        }
+        return row;
+    }
+
+    private void assertThirdAck(JSONObject observed) throws Exception {
+        assertEquals("Barrier must contain exactly three durable ACKs", 9, observed.getInt("pendingCount"));
+        assertEquals(0, observed.getInt("firstThreePending"));
+        assertEquals("pressure-004", observed.getString("firstRemaining"));
+        JSONObject ack = observed.getJSONObject("lastAcknowledgement");
+        assertEquals(201, ack.getInt("status"));
+        assertEquals("pressure-003", ack.getJSONObject("result").getJSONObject("item").getString("id"));
+        JSONObject original = originalPending.getJSONObject(2);
+        assertEquals(original.getString("client_mutation_id"), ack.getString("clientMutationId"));
+        assertEquals(original.getString("payload_hash"), ack.getString("payloadHash"));
+    }
+
+    private void startWatcher() {
+        watcher = new Thread(() -> {
+            try (io.requery.android.database.sqlite.SQLiteDatabase db = openItemObserver()) {
+                watcherReady.countDown();
+                long deadline = SystemClock.elapsedRealtime() + timeout("drainTimeoutMs");
+                int previousCount = -1;
+                while (!stopWatcher.get()) {
+                    JSONObject observed = ackObservation(db);
+                    int pending = observed.getInt("pendingCount");
+                    if (pending != previousCount) { event("ack-observation", observed); previousCount = pending; }
+                    assertTrue("Fourth ACK preceded the barrier observation", pending >= 9);
+                    if (pending == 9) {
+                        assertThirdAck(observed);
+                        barrierAt.set(observed.getLong("elapsedRealtime"));
+                        writeJson("barrier.json", observed);
+                        put("barrier", observed);
+                        event("barrier", observed);
+                        // ackObservation's cursor has already closed; no SQL lock
+                        // is held to keep production from issuing another request.
+                        requestNetwork("barrier-offline", false, false);
+                        event("work-cancel-request", object("workId", workId, "uniqueName", UNIQUE_WORK));
+                        WorkManager.getInstance(context).cancelUniqueWork(UNIQUE_WORK).getResult()
+                                .get(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS);
+                        event("work-cancel-operation-complete", object("workId", workId,
+                                "transportSettlementNotYetClaimed", true, "dispatcher", dispatcher()));
+                        observeNetwork("barrier-offline", false);
+                        // Raw native bytes are captured only after the main test
+                        // observes both transport and foreground settlement.
+                        return;
+                    }
+                    assertTrue("Three ACKs not observed before deadline", SystemClock.elapsedRealtime() < deadline);
+                    Thread.sleep(OBSERVE_MS);
+                }
+            } catch (Throwable error) {
+                watcherFailure.set(error);
+                try {
+                    put("watcherFailure", object("error", error.toString(), "stack", Log.getStackTraceString(error)));
+                } catch (Throwable recording) { error.addSuppressed(recording); }
+            } finally {
+                watcherReady.countDown();
+                barrierDone.countDown();
+            }
+        }, "M14-ACK-observer");
+        watcher.setDaemon(true);
+        watcher.start();
+    }
+
+    private void assertBeforeBarrier(String phase) {
+        assertEquals(phase + " occurred after the three-ACK barrier", 0, barrierAt.get());
+    }
+
+    private void nativeSchedule(int ordinal) throws Exception {
+        assertBeforeBarrier("native scheduling request " + ordinal);
+        CountDownLatch done = new CountDownLatch(1);
+        AtomicReference<Throwable> failed = new AtomicReference<>();
+        JSONObject request = object("ordinal", ordinal, "queuedAt", System.currentTimeMillis(),
+                "queuedElapsed", SystemClock.elapsedRealtime());
+        reactContext.runOnNativeModulesQueueThread(() -> {
+            try {
+                request.put("entryAt", System.currentTimeMillis()).put("entryElapsed", SystemClock.elapsedRealtime());
+                request.put("onNativeModulesQueue", reactContext.isOnNativeModulesQueueThread());
+                NativeModule registered = reactContext.getNativeModule("BackgroundSync");
+                if (registered == null) throw new AssertionError("Missing production BackgroundSync module");
+                assertTrue("Registered BackgroundSync must be the production module", registered instanceof BackgroundSyncModule);
+                BackgroundSyncModule module = (BackgroundSyncModule) registered;
+                assertEquals("BackgroundSync", module.getName());
+                request.put("moduleName", module.getName()).put("moduleInstanceIdentity", System.identityHashCode(module));
+                module.schedule(new PromiseImpl(values -> {
+                    try {
+                        request.put("status", "RESOLVED");
+                        request.put("value", new JSONObject(((ReadableMap) values[0]).toHashMap()));
+                    } catch (Throwable error) { failed.set(error); }
+                    finally {
+                        try {
+                            request.put("replyAt", System.currentTimeMillis()).put("replyElapsed", SystemClock.elapsedRealtime());
+                        } catch (Throwable error) { failed.compareAndSet(null, error); }
+                        done.countDown();
+                    }
+                }, values -> {
+                    try {
+                        request.put("status", "REJECTED").put("rejection",
+                                values.length > 0 && values[0] instanceof ReadableMap
+                                        ? new JSONObject(((ReadableMap) values[0]).toHashMap()) : Arrays.toString(values));
+                    } catch (Throwable error) { failed.set(error); }
+                    finally {
+                        failed.compareAndSet(null, new AssertionError("Production schedule rejected"));
+                        done.countDown();
+                    }
+                }));
+            } catch (Throwable error) { failed.set(error); done.countDown(); }
+        });
+        boolean completed = done.await(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS);
+        if (!completed) request.put("status", "TIMEOUT");
+        try {
+            request.put("work", workObservation());
+            request.put("workObservedAt", System.currentTimeMillis()).put("workObservedElapsed", SystemClock.elapsedRealtime());
+        } catch (Throwable error) {
+            request.put("workObservationError", error.toString());
+            failed.compareAndSet(null, error);
+        }
+        if (failed.get() != null) request.put("error", failed.get().toString());
+        synchronized (evidenceLock) { nativeRequests.put(request); save(); }
+        event("native-schedule", request);
+        assertTrue("Native schedule did not settle", completed);
+        if (failed.get() != null) throw new AssertionError("Native schedule failed", failed.get());
+        assertTrue(request.getBoolean("onNativeModulesQueue"));
+        assertEquals(cycleId, request.getJSONObject("value").getString("cycleId"));
+        assertBeforeBarrier("native scheduling reply " + ordinal);
+        JSONObject work = request.getJSONObject("work");
+        assertEquals(workId, work.getString("id"));
+        assertEquals(1, work.getInt("work_count"));
+        assertEquals("Initial drain must still be running", 1, work.getInt("state"));
+    }
+
+    private void foreground(String stage, Rect bounds, boolean burst) throws Exception {
+        if (burst) assertBeforeBarrier("foreground burst click");
+        assertTrue("Foreground request requires a ready real screen", hasDescription("Local storage ready"));
+        JSONObject request = object("stage", stage, "clickAt", System.currentTimeMillis(),
+                "clickElapsed", SystemClock.elapsedRealtime(), "bounds", bounds.toShortString());
+        synchronized (evidenceLock) { foregroundRequests.put(request); save(); }
+        assertTrue("Real Synchronize touch failed", device.click(bounds.centerX(), bounds.centerY()));
+        await(stage + " accepted foreground request", timeout("coordinatorTimeoutMs"),
+                () -> hasDescription("Sync status: refreshing"));
+        request.put("acceptedAt", System.currentTimeMillis()).put("acceptedElapsed", SystemClock.elapsedRealtime());
+        if (burst) {
+            assertBeforeBarrier("accepted foreground burst");
+            JSONObject work = workObservation();
+            request.put("work", work);
+            assertEquals(workId, work.getString("id"));
+            assertEquals("Foreground overlap must occur during the initial drain", 1, work.getInt("state"));
+        }
+        event("foreground-accepted", request);
+        save();
+    }
+
+    private void assertOriginalPending(JSONArray pending) throws Exception {
+        Map<String, JSONObject> originals = new HashMap<>();
+        for (int i = 0; i < originalPending.length(); i++) {
+            JSONObject row = originalPending.getJSONObject(i);
+            originals.put(row.getString("item_id"), row);
+        }
+        for (int i = 0; i < pending.length(); i++) {
+            JSONObject row = pending.getJSONObject(i);
+            JSONObject original = originals.get(row.getString("item_id"));
+            assertNotNull("Unexpected pending identity", original);
+            for (String key : new String[]{"sequence", "kind", "item_id", "payload", "client_mutation_id", "payload_hash"}) {
+                assertEquals("Pending intent changed: " + key, original.get(key), row.get(key));
+            }
+            assertTrue("Unexpected terminal error", row.isNull("terminal_error"));
+        }
+    }
+
+    private final Application.ActivityLifecycleCallbacks lifecycle = new Application.ActivityLifecycleCallbacks() {
+        private void record(String name, Activity current) {
+            if (!(current instanceof MainActivity)) return;
+            try {
+                event("activity-" + name, object("activityIdentity", System.identityHashCode(current),
+                        "applicationIdentity", System.identityHashCode(current.getApplication())));
+            } catch (Throwable failure) { watcherFailure.compareAndSet(null, failure); }
+        }
+        public void onActivityCreated(Activity a, Bundle state) { record("created", a); }
+        public void onActivityStarted(Activity a) { record("started", a); }
+        public void onActivityResumed(Activity a) { record("resumed", a); }
+        public void onActivityPaused(Activity a) { record("paused", a); }
+        public void onActivityStopped(Activity a) { record("stopped", a); }
+        public void onActivitySaveInstanceState(Activity a, Bundle state) { record("saved", a); }
+        public void onActivityDestroyed(Activity a) { record("destroyed", a); }
+    };
+
+    @Test
+    public void pressureSurvivesCancellationAndForegroundReconnect() throws Throwable {
+        assertTrue("Fresh evidence directory required", output.mkdir());
+        Throwable failure = null;
+        ActivityScenario<MainActivity> scenario = null;
+        try {
+            byte[] raw;
+            try (InputStream stream = instrumentation.getContext().getAssets().open("m14-inputs.json")) { raw = bytes(stream); }
+            input = new JSONObject(new String(raw, StandardCharsets.UTF_8));
+            write("inputs.json", raw);
+            put("inputSha256", sha(raw));
+            put("pid", pid);
+            put("applicationIdentity", System.identityHashCode(application));
+            put("processDeathClaim", false);
+            put("greedyEnabled", true);
+            assertEquals(34, Build.VERSION.SDK_INT);
+            assertEquals(12, input.getJSONArray("items").length());
+            assertEquals(4, input.getInt("nativeScheduleRequests"));
+            assertEquals(1, input.getInt("foregroundBurstRequests"));
+            assertEquals(1, input.getInt("foregroundReconnectRequests"));
+            assertEquals(3, input.getInt("barrierAcknowledgments"));
+            assertEquals(500, input.getInt("responseDelayMs"));
+            assertEquals(2, input.getInt("maxConcurrentMutations"));
+            assertFalse("No M10 scheduler suppression control may remain",
+                    new File(context.getFilesDir(), "m10-work-clock").exists());
+            put("initialNetwork", networkState());
+            assertTrue("External runner must establish actual offline before instrumentation", networkMatches(networkState(), false));
+            assertFalse("External runner must supply an empty local start", itemsFile.exists());
+            assertFalse("External runner must supply an empty scheduler", workFile.exists());
+            application.registerActivityLifecycleCallbacks(lifecycle);
+            scenario = ActivityScenario.launch(MainActivity.class);
+            scenario.onActivity(current -> activity = current);
+            // A nonzero root tag is allocated before its shadow root is
+            // registered. Do not enqueue a second runApplication until the
+            // initial real screen has rendered successfully.
+            ready();
+            assertTrue(hasDescription("Item count: 0"));
+            assertTrue(hasDescription("Pending uploads: 0"));
+            captureUi("initial-root-ready");
+            AtomicReference<JSONObject> initializedRoot = new AtomicReference<>();
+            onMain(() -> {
+                ReactRootView root = activity.getReactDelegate().getReactRootView();
+                assertNotNull(root);
+                assertTrue("Initial root must contain the rendered screen", root.getChildCount() > 0);
+                initializedRoot.set(object("rootTag", root.getRootViewTag(),
+                        "childCount", root.getChildCount(), "uiXml", "initial-root-ready.xml"));
+            });
+            event("initial-root-ready", initializedRoot.get());
+            onMain(() -> {
+                ReactRootView root = activity.getReactDelegate().getReactRootView();
+                assertNotNull(root);
+                Bundle properties = root.getAppProperties() == null ? new Bundle() : new Bundle(root.getAppProperties());
+                properties.putString("testIdentityPrefix", input.optString("identityPrefix"));
+                properties.remove("testMutationIdentity");
+                root.setAppProperties(properties);
+            });
+            event("identity-properties-applied", object("identityPrefix", input.getString("identityPrefix")));
+            // Ready here is a usability check, not an identity acknowledgment.
+            // The first real create below must persist pressure-001 before any
+            // dispatch; there is no warmup create, retry, or SQL correction.
+            ready();
+            onMain(() -> reactContext = application.getReactNativeHost().getReactInstanceManager().getCurrentReactContext());
+            assertNotNull(reactContext);
+            put("activityIdentity", System.identityHashCode(activity));
+            put("reactContextIdentity", System.identityHashCode(reactContext));
+            // RN 0.76.9 constructs this module with createClient(), not the
+            // provider singleton. Read the existing client; never replace it.
+            NetworkingModule networking = reactContext.getNativeModule(NetworkingModule.class);
+            assertNotNull(networking);
+            Field clientField = NetworkingModule.class.getDeclaredField("mClient");
+            clientField.setAccessible(true);
+            productionClient = (OkHttpClient) clientField.get(networking);
+            assertNotNull(productionClient);
+            put("dispatcherObservation", "Existing NetworkingModule.mClient; read-only pinned-field reflection");
+            clientObservation(true);
+            assertTrue(hasDescription("Item count: 0"));
+            assertTrue(hasDescription("Pending uploads: 0"));
+
+            for (int i = 0; i < 12; i++) {
+                JSONObject item = input.getJSONArray("items").getJSONObject(i);
+                UiObject2 title = control("New item title");
+                AtomicReference<JSONObject> editor = new AtomicReference<>();
+                onMain(() -> {
+                    View view = described(activity.getWindow().getDecorView(), "New item title");
+                    assertTrue("New item title must be the actual EditText", view instanceof EditText);
+                    EditText actual = (EditText) view;
+                    editor.set(object("text", actual.getText().toString(),
+                            "hint", actual.getHint() == null ? null : actual.getHint().toString(),
+                            "onUiThread", Looper.myLooper() == Looper.getMainLooper()));
+                });
+                event("editor-before-create", object("ordinal", i + 1, "editor", editor.get()));
+                assertEquals("Previous Save must clear its editor", "", editor.get().getString("text"));
+                title.click();
+                title.setText(item.getString("title"));
+                AtomicReference<JSONObject> imeHide = new AtomicReference<>();
+                onMain(() -> {
+                    assertFalse("Text entry must retain the existing Activity", activity.isFinishing());
+                    assertFalse(activity.isDestroyed());
+                    View window = activity.getWindow().getDecorView();
+                    assertNotNull(window.getWindowToken());
+                    InputMethodManager ime = (InputMethodManager) activity.getSystemService(Context.INPUT_METHOD_SERVICE);
+                    assertNotNull(ime);
+                    boolean accepted = ime.hideSoftInputFromWindow(window.getWindowToken(), 0);
+                    imeHide.set(object("activityIdentity", System.identityHashCode(activity),
+                            "windowIdentity", System.identityHashCode(window), "hideRequestAccepted", accepted,
+                            "finishing", activity.isFinishing(), "destroyed", activity.isDestroyed()));
+                });
+                // Record the request, not a keyboard-hidden claim. The same
+                // Add control is awaited below using its original timeout.
+                event("ime-hide-request", object("ordinal", i + 1, "window", imeHide.get()));
+                event("ui-create-request", object("ordinal", i + 1,
+                        "id", item.getString("id"), "title", item.getString("title")));
+                control("Add item").click();
+                final int count = i + 1;
+                await("production create " + count, timeout("uiTimeoutMs"),
+                        () -> hasDescription("Pending uploads: " + count) && hasDescription("Local storage ready"));
+                String committedId;
+                try (io.requery.android.database.sqlite.SQLiteDatabase db = openItemObserver();
+                     Cursor cursor = db.rawQuery("SELECT item_id FROM pending_mutations ORDER BY sequence DESC LIMIT 1", null)) {
+                    assertTrue(cursor.moveToFirst());
+                    committedId = cursor.getString(0);
+                    // A stale prop or lost UI callback fails; never repair IDs in SQL.
+                    assertEquals(item.getString("id"), committedId);
+                }
+                event("ui-create-committed", object("ordinal", count, "id", committedId));
+            }
+            ready();
+            JSONObject before = checkpoint("pre-dispatch", true);
+            JSONObject beforeTables = before.getJSONObject("items").getJSONObject("tables");
+            originalPending = beforeTables.getJSONArray("pending_mutations");
+            JSONArray local = beforeTables.getJSONArray("items");
+            assertEquals(12, originalPending.length());
+            assertEquals(12, local.length());
+            Set<String> identities = new HashSet<>();
+            for (int i = 0; i < 12; i++) {
+                JSONObject expected = input.getJSONArray("items").getJSONObject(i);
+                JSONObject pending = originalPending.getJSONObject(i);
+                assertEquals(expected.getString("id"), local.getJSONObject(i).getString("id"));
+                assertEquals(expected.getString("title"), local.getJSONObject(i).getString("title"));
+                assertEquals(0, local.getJSONObject(i).getInt("completed"));
+                assertEquals(expected.getString("id"), pending.getString("item_id"));
+                assertEquals("create", pending.getString("kind"));
+                JSONObject payload = new JSONObject(pending.getString("payload"));
+                assertEquals(expected.getString("id"), payload.getString("id"));
+                assertEquals(expected.getString("title"), payload.getString("title"));
+                assertFalse(payload.getBoolean("completed"));
+                assertEquals(input.getJSONArray("envelopes").getJSONObject(i).getString("payloadHash"), pending.getString("payload_hash"));
+                assertTrue("Distinct durable mutation identities required", identities.add(pending.getString("client_mutation_id")));
+                assertFalse(pending.getString("client_mutation_id").isEmpty());
+                assertEquals(0, pending.getInt("dispatched"));
+                assertTrue(pending.isNull("terminal_error"));
+            }
+            JSONObject metadata = beforeTables.getJSONArray("sync_metadata").getJSONObject(0);
+            assertTrue(metadata.isNull("last_acknowledgement"));
+            assertTrue(metadata.isNull("last_successful_refresh_at"));
+            put("originalPending", originalPending);
+            JSONObject registered = workObservation();
+            workId = registered.getString("id");
+            assertEquals(1, registered.getInt("work_count"));
+            assertEquals(0, registered.getInt("state"));
+            assertEquals(0, registered.getInt("run_attempt_count"));
+            cycleId = context.getSharedPreferences("item-background-sync", Context.MODE_PRIVATE).getString("cycleId", null);
+            assertNotNull(cycleId);
+            JSONObject workTables = before.getJSONObject("work").getJSONObject("tables");
+            JSONArray ids = workTables.getJSONArray("SystemIdInfo");
+            assertEquals(1, ids.length());
+            assertEquals(workId, ids.getJSONObject(0).getString("work_spec_id"));
+            int jobId = ids.getJSONObject(0).getInt("system_id");
+            put("workId", workId); put("cycleId", cycleId); put("jobId", jobId);
+            androidx.work.Configuration configuration = WorkManager.getInstance(context).getConfiguration();
+            put("schedulerConfiguration", object("clockClass", configuration.getClock().getClass().getName(),
+                    "runnableSchedulerClass", configuration.getRunnableScheduler().getClass().getName(),
+                    "clockValue", configuration.getClock().currentTimeMillis(), "m10ClockFileAbsent", true));
+            String jobs = shell("registered-jobs", "dumpsys", "jobscheduler", PACKAGE);
+            // The literal # form is API34's null/default namespace. Never pass -n null.
+            assertTrue("Observed WorkSpec must have a real default-namespace job",
+                    Pattern.compile("(?m)^\\s*JOB #[^/\\s]+/" + jobId + ": .*"
+                            + Pattern.quote(PACKAGE + "/" + SERVICE) + ".*$").matcher(jobs).find());
+            put("jobNamespace", JSONObject.NULL);
+            assertEquals(0, productionClient.dispatcher().runningCallsCount());
+            assertEquals(0, productionClient.dispatcher().queuedCallsCount());
+            Rect syncBounds = control("Synchronize").getVisibleBounds();
+            startWatcher();
+            assertTrue("ACK watcher did not attach", watcherReady.await(timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS));
+            if (watcherFailure.get() != null) throw new AssertionError("ACK watcher startup failed", watcherFailure.get());
+            changeNetwork("initial-online", true, true);
+            // Real OS callback even if Greedy already owns the same production drain.
+            String run = shell("nonforced-job-run", "cmd", "jobscheduler", "run", "-u", "0", PACKAGE, String.valueOf(jobId));
+            put("nonforcedJobRun", run);
+            assertTrue("Nonforced scheduler command did not accept the registered job", run.contains("Running job"));
+
+            AtomicInteger observations = new AtomicInteger();
+            await("actual live initial drain", timeout("drainTimeoutMs"), () -> {
+                assertBeforeBarrier("five-request burst start");
+                if (productionClient.dispatcher().runningCallsCount() == 0) return false;
+                JSONObject remote = fixtureState("first-live-" + observations.incrementAndGet());
+                JSONObject pressure = remote.getJSONObject("pressure");
+                JSONArray requests = pressure.getJSONArray("requests");
+                if (requests.length() == 0 || pressure.getInt("inFlightMutations") == 0) return false;
+                JSONObject observedWork = workObservation();
+                assertEquals(workId, observedWork.getString("id"));
+                assertEquals(1, observedWork.getInt("state"));
+                assertBeforeBarrier("observed active initial drain");
+                put("liveDrainAtBurstStart", object("fixture", remote, "work", observedWork,
+                        "observedAt", System.currentTimeMillis(), "observedElapsed", SystemClock.elapsedRealtime()));
+                return true;
+            });
+            assertEquals(1, workObservation().getInt("state"));
+            for (int i = 1; i <= 4; i++) nativeSchedule(i);
+            foreground("burst", syncBounds, true);
+            assertTrue("ACK barrier/cancellation did not finish", barrierDone.await(
+                    timeout("drainTimeoutMs") + timeout("networkTimeoutMs") * 3
+                            + timeout("coordinatorTimeoutMs"), TimeUnit.MILLISECONDS));
+            if (watcherFailure.get() != null) throw new AssertionError("ACK watcher failed", watcherFailure.get());
+            assertTrue("Barrier was not observed", barrierAt.get() > 0);
+            await("actual HTTP and foreground settlement", timeout("settlementTimeoutMs"),
+                    () -> productionClient.dispatcher().runningCallsCount() == 0
+                            && productionClient.dispatcher().queuedCallsCount() == 0
+                            && !hasDescription("Sync status: refreshing") && hasDescription("Local storage ready"));
+            event("transport-and-ui-settled", object("dispatcher", dispatcher(), "network", networkState()));
+            assertTrue(networkMatches(networkState(), false));
+            JSONObject settled = checkpoint("offline-settled", true);
+            put("barrierAfterOffline", object("checkpoint", "offline-settled",
+                    "items", settled.get("items"), "work", settled.get("work"),
+                    "preferences", settled.get("preferences"), "capturedAfterTransportSettlement", true));
+            JSONArray remaining = settled.getJSONObject("items").getJSONObject("tables").getJSONArray("pending_mutations");
+            assertEquals("A fourth ACK must not be hidden by the barrier", 9, remaining.length());
+            assertOriginalPending(remaining);
+            try (io.requery.android.database.sqlite.SQLiteDatabase db = openItemObserver()) {
+                assertThirdAck(ackObservation(db));
+            }
+            String initialLogs = shell("initial-drain-logcat", "logcat", "-d", "-v", "threadtime");
+            write("initial-drain-logcat.txt", initialLogs.getBytes(StandardCharsets.UTF_8));
+            boolean callback = false;
+            JSONArray dispatchLines = new JSONArray();
+            for (String line : initialLogs.split("\n")) {
+                if (line.contains(workId) && (line.contains("WM-SystemJobService")
+                        || line.contains("WM-GreedyScheduler") || line.contains("WM-Processor")
+                        || line.contains("MSEBackground"))) dispatchLines.put(line);
+                if (line.contains("WM-SystemJobService") && line.contains("onStartJob") && line.contains(workId)) callback = true;
+            }
+            put("dispatch", object("systemJobCallbackObserved", callback, "greedyEnabled", true,
+                    "owner", "See ordered scheduler/Processor lines; callback alone is not an ownership claim",
+                    "lines", dispatchLines, "logFile", "initial-drain-logcat.txt"));
+            assertTrue("Actual SystemJobService callback required", callback);
+            shell("after-cancel-jobs", "dumpsys", "jobscheduler", PACKAGE);
+            changeNetwork("reconnect-online", true, false);
+            foreground("reconnect", control("Synchronize").getVisibleBounds(), false);
+            try {
+                await("one foreground reconnect drain", timeout("drainTimeoutMs"),
+                        () -> hasDescription("Pending uploads: 0") && hasDescription("Sync status: fresh")
+                                && hasDescription("Local storage ready")
+                                && productionClient.dispatcher().runningCallsCount() == 0
+                                && productionClient.dispatcher().queuedCallsCount() == 0);
+            } catch (Throwable original) {
+                try { captureReconnectFailure(original); }
+                catch (Throwable ignored) { /* Diagnostic work cannot replace the original failure. */ }
+                throw original;
+            }
+            JSONObject finalCheckpoint = checkpoint("final", true);
+            JSONObject finalTables = finalCheckpoint.getJSONObject("items").getJSONObject("tables");
+            assertEquals(0, finalTables.getJSONArray("pending_mutations").length());
+            assertEquals(0, finalTables.getJSONArray("mutation_conflicts").length());
+            JSONArray finalItems = finalTables.getJSONArray("items");
+            assertEquals(12, finalItems.length());
+            Map<String, JSONObject> actual = new HashMap<>();
+            for (int i = 0; i < finalItems.length(); i++) {
+                JSONObject row = finalItems.getJSONObject(i);
+                assertNull("Duplicate visible Item", actual.put(row.getString("id"), row));
+            }
+            for (int i = 0; i < 12; i++) {
+                JSONObject expected = input.getJSONArray("items").getJSONObject(i);
+                JSONObject row = actual.get(expected.getString("id"));
+                assertNotNull(row);
+                assertEquals(expected.getString("title"), row.getString("title"));
+                assertEquals(0, row.getInt("completed"));
+                assertEquals(1, row.getInt("version"));
+                assertEquals(input.getLong("firstTimestamp") + i * 1000L, row.getLong("updated_at"));
+            }
+            assertFalse(finalTables.getJSONArray("sync_metadata").getJSONObject(0).isNull("last_successful_refresh_at"));
+            put("remoteFinal", fixtureState("final"));
+            assertEquals(4, nativeRequests.length());
+            assertEquals(2, foregroundRequests.length());
+            sameRuntime("finished");
+            put("status", "PASS");
+        } catch (Throwable error) {
+            failure = error;
+            put("status", "FAIL");
+            put("failure", object("error", error.toString(), "stack", Log.getStackTraceString(error)));
+        } finally {
+            stopWatcher.set(true);
+            if (watcher != null && watcher.isAlive()) {
+                watcher.interrupt();
+                watcher.join(input == null ? 1000 : timeout("networkTimeoutMs"));
+                if (watcher.isAlive()) {
+                    AssertionError alive = new AssertionError("ACK observer failed to stop");
+                    if (failure == null) failure = alive; else failure.addSuppressed(alive);
+                }
+            }
+            try {
+                if (reactContext != null) clientObservation(false);
+                writeJson("request-observations.json", requestObservation());
+                put("requestObservationFile", "request-observations.json");
+            } catch (Throwable recording) {
+                if (failure == null) failure = recording; else failure.addSuppressed(recording);
+            }
+            try {
+                if (input != null) shell("final-logcat", "logcat", "-d", "-v", "threadtime");
+            } catch (Throwable recording) {
+                if (failure == null) failure = recording; else failure.addSuppressed(recording);
+            }
+            if (scenario != null) scenario.close();
+            application.unregisterActivityLifecycleCallbacks(lifecycle);
+            shellExecutor.shutdownNow();
+            if (failure != null) {
+                put("status", "FAIL");
+                put("failure", object("error", failure.toString(), "stack", Log.getStackTraceString(failure)));
+            }
+            put("watcherStopped", watcher == null || !watcher.isAlive());
+            put("completedAt", System.currentTimeMillis());
+            save();
+        }
+        if (failure != null) throw failure;
+    }
+}
diff --git a/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt b/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt
index 517d2f6..c764801 100644
--- a/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt
+++ b/android/app/src/main/java/com/mse/reactnative/BackgroundSync.kt
@@ -1,6 +1,8 @@
 package com.mse.reactnative
 
 import android.content.Context
+import android.net.ConnectivityManager
+import android.net.Network
 import android.os.Handler
 import android.os.Looper
 import android.os.Process
@@ -284,6 +286,11 @@ class BackgroundWorker(context: Context, parameters: WorkerParameters) : Listena
 }
 
 class BackgroundSyncModule(context: ReactApplicationContext) : ReactContextBaseJavaModule(context) {
+    private val connectivity = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
+    private val application = context.applicationContext as MainApplication
+    private val foregroundNetworks = mutableMapOf<String, ConnectivityManager.NetworkCallback>()
+    private val lostForegroundNetworks = mutableSetOf<String>()
+    private var invalidated = false
     override fun getName() = "BackgroundSync"
     private fun answer(promise: Promise, action: () -> Any?) {
         try { promise.resolve(action()) } catch (error: Exception) { promise.reject("BACKGROUND_SYNC", error) }
@@ -295,6 +302,70 @@ class BackgroundSyncModule(context: ReactApplicationContext) : ReactContextBaseJ
     @ReactMethod fun reserve(token: String, promise: Promise) = answer(promise) { BackgroundSync.reserve(token) }
     @ReactMethod fun requestFinished(token: String, promise: Promise) = answer(promise) { BackgroundSync.requestFinished(token); null }
     @ReactMethod fun complete(token: String, outcome: String, promise: Promise) = answer(promise) { BackgroundSync.complete(token, outcome) }
+
+    @ReactMethod fun observeForegroundNetwork(token: String, promise: Promise) = answer(promise) {
+        synchronized(foregroundNetworks) {
+            check(!invalidated && !foregroundNetworks.containsKey(token))
+            val callback = object : ConnectivityManager.NetworkCallback() {
+                override fun onLost(network: Network) {
+                    synchronized(foregroundNetworks) {
+                        if (foregroundNetworks[token] !== this) return
+                        lostForegroundNetworks.add(token)
+                        application.evictIdleHttpConnections(token, "network_lost")
+                        Log.i("MSEForeground", JSONObject().put("phase", "NETWORK_LOST")
+                            .put("token", token).put("network", network.toString()).put("pid", Process.myPid()).toString())
+                        // Target this observer's runtime, never a replacement
+                        // React context or a background Worker token.
+                        if (reactApplicationContext.hasActiveReactInstance()) {
+                            reactApplicationContext.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
+                                .emit("ForegroundSyncOffline", Arguments.createMap().apply { putString("token", token) })
+                        }
+                    }
+                }
+            }
+            connectivity.registerDefaultNetworkCallback(callback)
+            foregroundNetworks[token] = callback
+            // Registration precedes this initial query. The already-installed
+            // JS listener also receives any loss racing the native response.
+            (connectivity.activeNetwork != null).also { connected ->
+                Log.i("MSEForeground", JSONObject().put("phase", "OBSERVE").put("token", token)
+                    .put("connected", connected).put("pid", Process.myPid()).toString())
+            }
+        }
+    }
+
+    @ReactMethod fun stopObservingForegroundNetwork(token: String, promise: Promise) = answer(promise) {
+        synchronized(foregroundNetworks) {
+            foregroundNetworks.remove(token)?.let { callback ->
+                try { connectivity.unregisterNetworkCallback(callback) }
+                finally {
+                    if (lostForegroundNetworks.remove(token)) {
+                        application.evictIdleHttpConnections(token, "observer_stop")
+                    }
+                }
+                Log.i("MSEForeground", JSONObject().put("phase", "STOP").put("token", token)
+                    .put("pid", Process.myPid()).toString())
+            }
+        }
+        null
+    }
+
+    override fun invalidate() {
+        synchronized(foregroundNetworks) {
+            invalidated = true
+            val callbacks = foregroundNetworks.toMap()
+            foregroundNetworks.clear()
+            callbacks.forEach { (token, callback) ->
+                try { connectivity.unregisterNetworkCallback(callback) }
+                catch (error: Exception) { Log.e("MSEForeground", "Network observer cleanup failed", error) }
+                if (lostForegroundNetworks.remove(token)) {
+                    application.evictIdleHttpConnections(token, "module_invalidate")
+                }
+            }
+            lostForegroundNetworks.clear()
+        }
+        super.invalidate()
+    }
 }
 
 class BackgroundSyncPackage : ReactPackage {
diff --git a/android/app/src/main/java/com/mse/reactnative/MainApplication.kt b/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
index 75fe170..26184cc 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainApplication.kt
@@ -1,6 +1,8 @@
 package com.mse.reactnative
 
 import android.app.Application
+import android.os.Process
+import android.os.SystemClock
 import android.util.Log
 import androidx.work.Clock
 import androidx.work.Configuration
@@ -16,8 +18,32 @@ import com.facebook.react.soloader.OpenSourceMergedSoMapping
 import com.facebook.react.modules.network.OkHttpClientProvider
 import com.facebook.soloader.SoLoader
 import java.io.File
+import okhttp3.ConnectionPool
+import org.json.JSONObject
 
 class MainApplication : Application(), ReactApplication, Configuration.Provider {
+    // NetworkingModule uses createClient(), not the provider singleton. Supply
+    // the pool through that actual factory so loss cleanup reaches its sockets.
+    private val networkingPool = ConnectionPool()
+
+    internal fun evictIdleHttpConnections(token: String, reason: String) {
+        try {
+            val idleBefore = networkingPool.idleConnectionCount()
+            val totalBefore = networkingPool.connectionCount()
+            networkingPool.evictAll() // Idle entries only; never cancels an active call.
+            Log.i("MSEForeground", JSONObject().put("phase", "IDLE_CONNECTIONS_EVICTED")
+                .put("token", token).put("reason", reason).put("pid", Process.myPid())
+                .put("poolIdentity", System.identityHashCode(networkingPool))
+                .put("idleBefore", idleBefore).put("totalBefore", totalBefore)
+                .put("idleAfter", networkingPool.idleConnectionCount())
+                .put("totalAfter", networkingPool.connectionCount())
+                .put("elapsed", SystemClock.elapsedRealtime()).toString())
+        } catch (error: Exception) {
+            // A cleanup error must not suppress the existing JS loss signal.
+            Log.e("MSEForeground", "Idle HTTP pool cleanup failed: $reason", error)
+        }
+    }
+
     override val reactNativeHost: ReactNativeHost = object : DefaultReactNativeHost(this) {
         override fun getPackages(): List<ReactPackage> = PackageList(this).packages.apply { add(BackgroundSyncPackage()) }
         override fun getJSMainModuleName(): String = "index"
@@ -54,7 +80,7 @@ class MainApplication : Application(), ReactApplication, Configuration.Provider
         // durable allowance. Foreground still uses the same fetch/ACK protocol.
         OkHttpClientProvider.setOkHttpClientFactory {
             OkHttpClientProvider.createClientBuilder().retryOnConnectionFailure(false)
-                .followRedirects(false).followSslRedirects(false).build()
+                .followRedirects(false).followSslRedirects(false).connectionPool(networkingPool).build()
         }
     }
 }
diff --git a/fixture/server.cjs b/fixture/server.cjs
index ec9f9e2..24d7b49 100644
--- a/fixture/server.cjs
+++ b/fixture/server.cjs
@@ -1,4 +1,4 @@
-// A disposable deterministic M03–M10 fixture, not a backend service.
+// A disposable deterministic phase-1 fixture, not a backend service.
 const http = require('node:http');
 const {createHash} = require('node:crypto');
 
@@ -28,6 +28,7 @@ function createFixture() {
   let tombstones;
   let m10Case = null;
   let m10HttpAttempts = 0;
+  let pressure = null;
   let heldResponse = null;
   let holdTimer = null;
   let hold = {phase: 'IDLE', maxHoldMs: 30000};
@@ -58,11 +59,17 @@ function createFixture() {
     tombstones = new Map();
     m10Case = null;
     m10HttpAttempts = 0;
+    pressure = null;
   }
   reset();
   const deletionState = () => tombstones.size ? {tombstones: deletedSnapshot()} : {};
   const state = () => ({items: snapshot(), ...deletionState(), nextTimestamp, requests});
   const tick = () => {const timestamp = nextTimestamp; nextTimestamp += 1000; return timestamp;};
+  const pressureState = () => pressure === null ? null : {
+    delayMs: pressure.delayMs, inFlightMutations: pressure.inFlightMutations,
+    peakInFlightMutations: pressure.peakInFlightMutations,
+    requests: pressure.requests.map(entry => ({...entry})),
+  };
 
   const server = http.createServer(async (request, response) => {
     let body;
@@ -72,9 +79,38 @@ function createFixture() {
     const {method, url: path} = request;
     const m10Attempt = m10Case !== null && !path.startsWith('/__') ? ++m10HttpAttempts : null;
     const receivedAt = m10Attempt === null ? null : Date.now();
+    // Count real socket lifetimes, not handlers waiting on a server-side lock.
+    // Each response has its own timer; controls and other requests keep running.
+    const measurement = pressure !== null && !path.startsWith('/__') ? pressure : null;
+    let pressureEntry = null;
+    let pressureTimer = null;
+    if (measurement !== null) {
+      const isMutation = method === 'POST' || method === 'PATCH' || method === 'DELETE';
+      pressureEntry = {requestId: measurement.requests.length + 1, method, path, isMutation,
+        receivedAt: Date.now(), receivedMonotonicNs: process.hrtime.bigint().toString(),
+        finishedAt: null, disconnectedAt: null, endedAt: null};
+      measurement.requests.push(pressureEntry);
+      if (isMutation) {
+        measurement.inFlightMutations += 1;
+        measurement.peakInFlightMutations = Math.max(measurement.peakInFlightMutations, measurement.inFlightMutations);
+      }
+      const settle = phase => {
+        pressureEntry[phase] = Date.now();
+        if (pressureTimer !== null) {clearTimeout(pressureTimer); pressureTimer = null;}
+        if (pressureEntry.endedAt === null) {
+          pressureEntry.endedAt = Date.now();
+          pressureEntry.endedMonotonicNs = process.hrtime.bigint().toString();
+          if (isMutation) {measurement.inFlightMutations -= 1;}
+        }
+      };
+      response.once('finish', () => settle('finishedAt'));
+      response.once('close', () => settle('disconnectedAt'));
+      request.once('aborted', () => {pressureEntry.requestAbortedAt = Date.now();});
+    }
     function reply(status, payload, outcome = 'rejected') {
       if (!path.startsWith('/__')) {requests.push({method, path, body: body ?? null, status, response: payload,
-        ...(m10Attempt === null ? {} : {m10Attempt, receivedAt})});}
+        ...(m10Attempt === null ? {} : {m10Attempt, receivedAt}),
+        ...(pressureEntry === null ? {} : {pressureRequestId: pressureEntry.requestId})});}
       const applied = mutation && status >= 200 && status < 300 && outcome !== 'duplicate';
       if (applied) {
         mutationEvidence.appliedCount += 1;
@@ -88,7 +124,22 @@ function createFixture() {
         || (dropMutationId !== null && clientMutationId === dropMutationId)));
       const responseHeld = applied && hold.phase === 'ARMED' && clientMutationId === hold.clientMutationId;
       if (mutation) {mutationEvidence.attempts.push({...mutation, outcome, status, response: payload, responseDropped,
-        ...(responseHeld ? {responseHeld: true} : {})});}
+        ...(responseHeld ? {responseHeld: true} : {}),
+        ...(pressureEntry === null ? {} : {pressureRequestId: pressureEntry.requestId})});}
+      if (pressureEntry !== null) {
+        Object.assign(pressureEntry, {wireBody, status, outcome, response: payload,
+          ...(mutation ? {clientMutationId, canonical: mutation.canonical, payloadHash: mutation.actualHash} : {}),
+          appliedAt: applied ? Date.now() : null, delayMs: measurement.delayMs, scheduledAt: Date.now()});
+        pressureTimer = setTimeout(() => {
+          pressureTimer = null;
+          pressureEntry.timerAt = Date.now();
+          if (response.destroyed) {return;}
+          pressureEntry.headersAt = Date.now();
+          response.writeHead(status, {'Content-Type': 'application/json'});
+          response.end(JSON.stringify(payload));
+        }, measurement.delayMs);
+        return;
+      }
       if (responseHeld) {
         // Item, timestamp, receipt and applied evidence above are committed before
         // this barrier is observable. Controls keep serving on separate requests.
@@ -121,6 +172,14 @@ function createFixture() {
       if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
       if (method === 'GET' && path === '/__state') {return reply(200, state());}
       if (method === 'GET' && path === '/__mutation-state') {return reply(200, mutationEvidence);}
+      if (method === 'POST' && path === '/__m14-reset') {
+        reset();
+        items.clear();
+        nextTimestamp = 1700001000000;
+        pressure = {delayMs: 500, inFlightMutations: 0, peakInFlightMutations: 0, requests: []};
+        return reply(200, {...state(), pressure: pressureState()});
+      }
+      if (method === 'GET' && path === '/__m14-state') {return reply(200, {...state(), pressure: pressureState()});}
       if (method === 'POST' && path === '/__m10-reset') {
         if (body?.case !== 'A' && body?.case !== 'B') {return reply(400, {error: 'M10 case A or B required'});}
         reset();
@@ -291,7 +350,7 @@ function createFixture() {
       return reply(400, {error: 'Invalid JSON request'});
     }
   });
-  return {server, reset, state, mutationState: () => mutationEvidence};
+  return {server, reset, state, mutationState: () => mutationEvidence, pressureState};
 }
 
 module.exports = {createFixture};
diff --git a/scripts/verify_m14.py b/scripts/verify_m14.py
new file mode 100644
index 0000000..5177b31
--- /dev/null
+++ b/scripts/verify_m14.py
@@ -0,0 +1,483 @@
+#!/usr/bin/env python3
+"""One root-authorized M14 fixed12 invocation on actual Android.
+
+The native test is attached before UI setup. This wrapper never creates Items,
+calls a Worker, dispatches substitute work, or retries the complete scenario.
+"""
+import argparse
+import base64
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
+
+PACKAGE = "com.mse.reactnative"
+TEST = PACKAGE + ".M14PressureTest"
+URL = "http://127.0.0.1:18081"
+NETWORK_KEYS = ("airplane_mode_on", "wifi_on", "mobile_data")
+ONLINE = dict(zip(NETWORK_KEYS, ("0", "1", "1")))
+OFFLINE = dict(zip(NETWORK_KEYS, ("1", "0", "0")))
+M10_APK_SHA256 = "7f8c1110bc38ca195d1572d6b419a9e5a3dc97cb5441df208aa70900fe8b5c27"
+
+
+def sha(path):
+    return hashlib.sha256(Path(path).read_bytes()).hexdigest()
+
+
+def native_database(path, analysis):
+    """Query a separate copy so SQLite cannot alter the preserved raw WAL/SHM."""
+    analysis.mkdir()
+    files = [Path(str(path) + suffix) for suffix in ("", "-wal", "-shm")]
+    hashes = {file.name: sha(file) for file in files if file.exists()}
+    for file in files:
+        if file.exists():
+            shutil.copy2(file, analysis / file.name)
+    with sqlite3.connect(f"file:{analysis / path.name}?mode=ro", uri=True) as db:
+        db.row_factory = sqlite3.Row
+        assert [row[0] for row in db.execute("PRAGMA integrity_check")] == ["ok"], path
+        tables = {}
+        names = [row[0] for row in db.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
+        for name in names:
+            assert re.fullmatch(r"[A-Za-z_][A-Za-z_0-9]*", name)
+            tables[name] = [dict(row) for row in db.execute('SELECT * FROM "' + name + '" ORDER BY rowid')]
+        value = {"schemaVersion": db.execute("PRAGMA user_version").fetchone()[0], "tables": tables}
+    assert all(sha(path.parent / name) == digest for name, digest in hashes.items())
+    # WorkManager Data is binary; retain exact bytes, not a guessed text decoder.
+    return json.loads(json.dumps(value, default=lambda blob: {"base64": base64.b64encode(blob).decode()})), hashes
+
+
+def audit_fixture(state, mutations, inputs):
+    pressure = state["pressure"]
+    assert pressure["delayMs"] == inputs["responseDelayMs"] == 500
+    assert pressure["inFlightMutations"] == 0
+    assert 1 <= pressure["peakInFlightMutations"] <= inputs["maxConcurrentMutations"] == 2
+    expected = [{**item, "version": 1, "updatedAt": inputs["firstTimestamp"] + index * 1000}
+                for index, item in enumerate(inputs["items"])]
+    assert state["items"] == expected
+    assert state["nextTimestamp"] == inputs["finalNextTimestamp"] == 1700001012000
+    assert state.get("tombstones", []) == []
+    assert mutations["appliedCount"] == 12
+    assert mutations["conflictCount"] == mutations["hashRejectedCount"] == 0
+    envelopes = {entry["itemId"]: entry for entry in inputs["envelopes"]}
+    identities, intervals, applied = {}, [], []
+    measurements = {entry["requestId"]: entry for entry in pressure["requests"]}
+    assert len(measurements) == len(pressure["requests"])
+    for entry in pressure["requests"]:
+        assert entry["delayMs"] == 500 and entry["endedAt"] is not None, entry
+        assert entry["endedAt"] >= entry["receivedAt"]
+        if entry["finishedAt"] is not None:
+            assert entry["headersAt"] >= entry["scheduledAt"]
+            assert entry["finishedAt"] >= entry["headersAt"]
+        if entry["isMutation"]:
+            assert entry["method"] == "POST" and entry["path"] == "/items"
+            wire = entry["wireBody"]
+            item_id = wire["id"]
+            frozen = envelopes[item_id]
+            payload = {key: value for key, value in wire.items() if key not in ("clientMutationId", "payloadHash")}
+            actual_canonical = json.dumps({"method": "POST", "path": "/items", "payload": payload}, sort_keys=True, separators=(",", ":"))
+            assert actual_canonical == frozen["canonical"]
+            assert entry["canonical"] == frozen["canonical"]
+            assert entry["payloadHash"] == wire["payloadHash"] == frozen["payloadHash"]
+            assert hashlib.sha256(entry["canonical"].encode()).hexdigest() == wire["payloadHash"]
+            assert wire["clientMutationId"] and entry["clientMutationId"] == wire["clientMutationId"]
+            if item_id in identities:
+                assert identities[item_id] == wire["clientMutationId"]
+            identities[item_id] = wire["clientMutationId"]
+            assert entry["status"] == 201 and entry["outcome"] in ("applied", "duplicate")
+            if entry["outcome"] == "applied":
+                applied.append(item_id)
+            intervals.extend(((int(entry["receivedMonotonicNs"]), 1), (int(entry["endedMonotonicNs"]), -1)))
+        else:
+            assert entry["method"] == "GET" and entry["path"] == "/items" and entry["status"] == 200
+    assert applied == [item["id"] for item in inputs["items"]]
+    assert len(set(identities.values())) == 12
+    active = peak = 0
+    for _, change in sorted(intervals):
+        active += change
+        assert active >= 0
+        peak = max(peak, active)
+    assert active == 0 and peak == pressure["peakInFlightMutations"]
+    for attempt in mutations["attempts"]:
+        measured = measurements[attempt["pressureRequestId"]]
+        assert attempt["wireBody"] == measured["wireBody"]
+        assert attempt["canonical"] == measured["canonical"]
+        assert attempt["status"] == measured["status"] and attempt["outcome"] == measured["outcome"]
+    assert len(mutations["attempts"]) == 12 + mutations["duplicateCount"]
+    return {"canonicalItems": expected, "identities": identities, "peak": peak,
+            "applied": 12, "duplicates": mutations["duplicateCount"], "mutationRequests": len(mutations["attempts"])}
+
+
+def audit_native(native_dir, native, inputs, fixture):
+    assert native["inputSha256"] == sha(native_dir / "inputs.json")
+    assert json.loads((native_dir / "inputs.json").read_text()) == inputs
+    assert native["pid"] > 0 and native["processDeathClaim"] is False
+    assert native["greedyEnabled"] and native["watcherStopped"]
+    assert native["schedulerConfiguration"]["m10ClockFileAbsent"] is True
+    events = json.loads((native_dir / "events.json").read_text())
+    assert all(event["pid"] == native["pid"] for event in events)
+    creates = [event for event in events if event["event"] == "ui-create-request"]
+    commits = [event for event in events if event["event"] == "ui-create-committed"]
+    assert len(creates) == len(commits) == 12
+    for index, (create, commit) in enumerate(zip(creates, commits)):
+        assert create["ordinal"] == commit["ordinal"] == index + 1
+        assert create["id"] == commit["id"] == inputs["items"][index]["id"]
+        assert create["title"] == inputs["items"][index]["title"]
+        assert create["elapsed"] <= commit["elapsed"]
+    runtime = [event for event in events if event["event"] == "runtime"]
+    assert {event["label"] for event in runtime} == {"pre-dispatch", "offline-settled", "final", "finished"}
+    for event in runtime:
+        for field in ("applicationIdentity", "activityIdentity", "reactContextIdentity"):
+            assert event[field] == native[field]
+
+    points = native["checkpoints"]
+    assert set(points) == {"pre-dispatch", "offline-settled", "final"}
+    databases = {}
+    for name, point in points.items():
+        databases[name] = {}
+        for kind in ("items", "work"):
+            declared = point[kind]
+            assert Path(declared["baseFile"]).name == declared["baseFile"]
+            reported = json.loads((native_dir / declared["jsonFile"]).read_text())
+            assert reported == declared
+            actual, hashes = native_database(native_dir / declared["baseFile"], native_dir.parent / (name + "-" + kind + "-analysis"))
+            assert actual == {key: declared[key] for key in ("schemaVersion", "tables")}, (name, kind)
+            assert hashes == declared["rawSha256"]
+            databases[name][kind] = actual["tables"]
+        preferences = point["preferences"]
+        assert sha(native_dir / preferences["file"]) == preferences["sha256"]
+        pref_values = {}
+        for node in ET.parse(native_dir / preferences["file"]).getroot():
+            pref_values[node.get("name")] = node.text if node.tag == "string" else int(node.get("value"))
+        assert pref_values == preferences["values"]
+        unfinished = [work for work in point["uniqueWork"] if work["state"] not in ("SUCCEEDED", "FAILED", "CANCELLED")]
+        assert len(unfinished) <= 1, point["uniqueWork"]
+        assert point["dispatcher"]["runningCount"] == point["dispatcher"]["queuedCount"] == 0
+        expected_network = ONLINE if name == "final" else OFFLINE
+        assert {key: point["network"][key] for key in NETWORK_KEYS} == expected_network
+        assert point["network"]["connected"] == (name == "final")
+        labels = {node.get("content-desc") for node in ET.parse(native_dir / point["uiXml"]).iter("node")}
+        assert "Item count: 12" in labels and "Local storage ready" in labels
+        assert "Pending uploads: " + str({"pre-dispatch": 12, "offline-settled": 9, "final": 0}[name]) in labels
+
+    before, settled, final = (databases[name]["items"] for name in ("pre-dispatch", "offline-settled", "final"))
+    original = before["pending_mutations"]
+    assert len(original) == len(before["items"]) == 12
+    assert native["originalPending"] == original
+    for index, operation in enumerate(original):
+        item = inputs["items"][index]
+        assert operation["sequence"] == index + 1 and operation["kind"] == "create"
+        assert operation["item_id"] == item["id"] and json.loads(operation["payload"]) == item
+        assert operation["payload_hash"] == inputs["envelopes"][index]["payloadHash"]
+        assert operation["client_mutation_id"] == fixture["identities"][item["id"]]
+        assert operation["dispatched"] == 0 and operation["terminal_error"] is None
+        assert before["items"][index] == {"id": item["id"], "title": item["title"], "completed": 0,
+                                           "version": 1, "updated_at": before["items"][index]["updated_at"]}
+    assert before["remote_versions"] == before["mutation_conflicts"] == []
+    assert before["sync_metadata"] == [{"singleton": 1, "last_successful_refresh_at": None, "last_acknowledgement": None}]
+    assert len(set(operation["client_mutation_id"] for operation in original)) == 12
+    assert [row["item_id"] for row in settled["pending_mutations"]] == [row["item_id"] for row in original[3:]]
+    for row, old in zip(settled["pending_mutations"], original[3:]):
+        assert {key: value for key, value in row.items() if key != "dispatched"} == {key: value for key, value in old.items() if key != "dispatched"}
+        assert row["dispatched"] in (0, 1)
+    for table in (before, settled, final):
+        assert table["local_identity"] == [{"singleton": 1, "next_id": 13}]
+        assert table["mutation_conflicts"] == []
+        assert table["sqlite_sequence"] == [{"name": "pending_mutations", "seq": 12}]
+    barrier = json.loads((native_dir / "barrier.json").read_text())
+    # Events add observation metadata to the same object; compare the actual SQL fields.
+    for key in ("pendingCount", "firstThreePending", "firstRemaining", "last_acknowledgement", "lastAcknowledgement", "observedAt", "elapsedRealtime"):
+        assert barrier[key] == native["barrier"][key]
+    assert barrier["pendingCount"] == 9 and barrier["firstThreePending"] == 0 and barrier["firstRemaining"] == "pressure-004"
+    expected_ack = {"clientMutationId": original[2]["client_mutation_id"], "payloadHash": original[2]["payload_hash"],
+                    "status": 201, "result": {"item": fixture["canonicalItems"][2]}}
+    assert json.loads(barrier["last_acknowledgement"]) == barrier["lastAcknowledgement"] == expected_ack
+    assert json.loads(settled["sync_metadata"][0]["last_acknowledgement"]) == expected_ack
+    assert settled["sync_metadata"][0]["last_successful_refresh_at"] is None
+    expected_local = [{"id": item["id"], "title": item["title"], "completed": 0, "version": 1, "updated_at": item["updatedAt"]}
+                      for item in fixture["canonicalItems"]]
+    assert settled["items"] == expected_local[:3] + before["items"][3:]
+    assert sorted(final["items"], key=lambda row: row["id"]) == expected_local
+    assert final["pending_mutations"] == []
+    assert len(settled["remote_versions"]) == 3 and len(final["remote_versions"]) == 12
+    assert json.loads(final["sync_metadata"][0]["last_acknowledgement"]) == {
+        "clientMutationId": original[-1]["client_mutation_id"], "payloadHash": original[-1]["payload_hash"],
+        "status": 201, "result": {"item": fixture["canonicalItems"][-1]}}
+    assert native["barrierAfterOffline"]["checkpoint"] == "offline-settled"
+    assert native["barrierAfterOffline"]["capturedAfterTransportSettlement"] is True
+    for kind in ("items", "work", "preferences"):
+        assert native["barrierAfterOffline"][kind] == points["offline-settled"][kind]
+
+    calls = native["nativeScheduleRequests"]
+    live = native["liveDrainAtBurstStart"]
+    assert live["work"]["id"] == native["workId"] and live["work"]["state"] == 1
+    assert live["fixture"]["pressure"]["inFlightMutations"] >= 1
+    assert len(calls) == 4 and [call["ordinal"] for call in calls] == [1, 2, 3, 4]
+    for call in calls:
+        assert call["status"] == "RESOLVED" and call["onNativeModulesQueue"]
+        assert call["value"]["cycleId"] == native["cycleId"]
+        assert call["entryElapsed"] <= call["replyElapsed"] <= barrier["elapsedRealtime"]
+        assert call["work"]["id"] == native["workId"] and call["work"]["state"] == 1 and call["work"]["work_count"] == 1
+    foreground = native["foregroundRequests"]
+    assert [call["stage"] for call in foreground] == ["burst", "reconnect"]
+    assert foreground[0]["clickElapsed"] <= foreground[0]["acceptedElapsed"] <= barrier["elapsedRealtime"]
+    assert foreground[0]["work"]["id"] == native["workId"] and foreground[0]["work"]["state"] == 1
+    assert foreground[1]["clickElapsed"] <= foreground[1]["acceptedElapsed"]
+    assert final["sync_metadata"][0]["last_successful_refresh_at"] >= foreground[1]["clickAt"]
+
+    required = ("network-request", "work-cancel-request", "work-cancel-operation-complete", "transport-and-ui-settled")
+    cancellation = [event for event in events if event["event"] in required and
+                    (event["event"] != "network-request" or event["label"] == "barrier-offline")]
+    assert [event["event"] for event in cancellation] == list(required)
+    assert barrier["elapsedRealtime"] <= cancellation[0]["elapsed"] <= cancellation[-1]["elapsed"]
+    assert cancellation[-1]["dispatcher"]["runningCount"] == cancellation[-1]["dispatcher"]["queuedCount"] == 0
+    assert cancellation[-1]["elapsed"] <= foreground[1]["clickElapsed"]
+    job_calls = [event for event in events if event["event"] == "shell" and event["label"] == "nonforced-job-run"]
+    assert len(job_calls) == 1
+    assert job_calls[0]["argv"] == ["cmd", "jobscheduler", "run", "-u", "0", PACKAGE, str(native["jobId"])]
+    assert native["jobNamespace"] is None and "Running job" in native["nonforcedJobRun"]
+    dispatch = native["dispatch"]
+    logs = (native_dir / dispatch["logFile"]).read_text()
+    assert dispatch["systemJobCallbackObserved"] and dispatch["greedyEnabled"]
+    assert any("WM-SystemJobService" in line and "onStartJob" in line and native["workId"] in line for line in logs.splitlines())
+    assert all(line in logs for line in dispatch["lines"])
+    work = databases["pre-dispatch"]["work"]
+    assert len(work["WorkSpec"]) == 1 and work["WorkSpec"][0]["id"] == native["workId"]
+    assert work["WorkSpec"][0]["state"] == 0 and work["WorkSpec"][0]["required_network_type"] == 1
+    assert work["SystemIdInfo"][0]["system_id"] == native["jobId"]
+    return {"databaseCount": len(databases) * 2, "barrierPending": 9, "finalPending": 0,
+            "pid": native["pid"], "workId": native["workId"], "cycleId": native["cycleId"],
+            "nativeBurstRequests": len(calls), "foregroundBurstRequests": 1, "foregroundReconnectRequests": 1}
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--node", default="node")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--test-apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    parser.add_argument("--baseline", action="store_true")
+    args = parser.parse_args()
+    root = Path(__file__).resolve().parent.parent
+    evidence = Path(args.evidence).resolve()
+    evidence.mkdir(parents=True, exist_ok=False)
+    inputs_file = root / "verification/M14-inputs.json"
+    inputs = json.loads(inputs_file.read_text())
+    assert inputs_file.read_bytes() == (root / "android/app/src/androidTest/assets/m14-inputs.json").read_bytes()
+    commands, controls = [], []
+    result = {"status": "RUNNING", "baseline": args.baseline, "hostPid": os.getpid(),
+              "apkSha256": sha(args.apk), "testApkSha256": sha(args.test_apk),
+              "harnessSha256": sha(__file__), "inputsSha256": sha(inputs_file),
+              "fixtureSha256": sha(root / "fixture/server.cjs"),
+              "resetPredicateSha256": sha(root / "scripts/verify_m07.py"),
+              "invocationPolicy": "One root-charged fixed12 case; no automatic retries or warmups"}
+    fixture, original_network, suite = None, None, None
+    started = time.monotonic()
+
+    def save():
+        (evidence / "result.json").write_text(json.dumps(result, indent=2) + "\n")
+
+    def adb(label, *parts, check=True, binary=False, timeout=60):
+        command = [args.adb, "-s", args.serial, *parts]
+        entry = {"label": label, "command": command, "timeoutSeconds": timeout,
+                 "startedAt": int(time.time() * 1000)}
+        before = time.monotonic()
+        try:
+            completed = subprocess.run(command, capture_output=True, timeout=timeout)
+            raw, err = completed.stdout, completed.stderr
+            entry["exit"] = completed.returncode
+        except subprocess.TimeoutExpired as error:
+            raw, err = error.stdout or b"", error.stderr or b""
+            entry.update(exit=None, error=repr(error))
+        for stream, data in (("stdout", raw), ("stderr", err)):
+            name = f"adb-{len(commands):04d}-{label}.{stream}"
+            (evidence / name).write_bytes(data)
+            entry[stream] = name
+        entry.update(elapsedSeconds=time.monotonic() - before, endedAt=int(time.time() * 1000))
+        commands.append(entry)
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2) + "\n")
+        assert entry["exit"] is not None, entry
+        if check:
+            assert entry["exit"] == 0, entry
+        return raw if binary else raw.decode(errors="replace").strip()
+
+    def remote(path, body=None):
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
+        deadline = time.monotonic() + inputs["networkTimeoutMs"] / 1000
+        while True:
+            settings = {key: adb(label + "-" + key, "shell", "settings", "get", "global", key) for key in NETWORK_KEYS}
+            dump = adb(label + "-connectivity", "shell", "dumpsys", "connectivity")
+            offline = "Active default network: none" in dump
+            if expected is None or (settings == expected and offline == (expected == OFFLINE)):
+                assert offline == (settings == OFFLINE)
+                return settings
+            assert time.monotonic() < deadline, (expected, settings)
+            time.sleep(0.1)
+
+    def quiet_reset():
+        begin, quiet = time.monotonic(), None
+        observations = result["resetObservations"] = []
+        while True:
+            left = 10 - (time.monotonic() - begin)
+            assert left > 0, "Initial Activity teardown did not settle within 10s"
+            pid = adb("setup-pid", "shell", "pidof", PACKAGE, check=False, timeout=left)
+            assert commands[-1]["exit"] in (0, 1)
+            left = 10 - (time.monotonic() - begin)
+            assert left > 0
+            activities = adb("setup-activities", "shell", "dumpsys", "activity", "activities", timeout=left)
+            present = package_in_live_activities(activities)
+            now = time.monotonic()
+            observations.append({"elapsedSeconds": now - begin, "pid": pid, "liveActivity": present,
+                                 "pidCommandIndex": len(commands) - 2, "activityCommandIndex": len(commands) - 1})
+            save()
+            assert now - begin < 10
+            if not pid and not present:
+                quiet = now if quiet is None else quiet
+                if now - quiet >= 1:
+                    return
+            else:
+                quiet = None
+            time.sleep(0.1)
+
+    def collect_native():
+        archive = adb("native-evidence", "exec-out", "run-as", PACKAGE, "tar", "-cf", "-", "files/m14-evidence", binary=True, check=False)
+        (evidence / "native-evidence.tar").write_bytes(archive)
+        assert commands[-1]["exit"] == 0
+        native_dir = evidence / "native"
+        native_dir.mkdir()
+        with tarfile.open(fileobj=io.BytesIO(archive)) as saved:
+            for member in saved:
+                if member.isdir():
+                    continue
+                assert member.isfile() and member.name.startswith("files/m14-evidence/")
+                relative = Path(member.name).relative_to("files/m14-evidence")
+                assert len(relative.parts) == 1 and relative.name not in (".", "..")
+                (native_dir / relative).write_bytes(saved.extractfile(member).read())
+        native = json.loads((native_dir / "result.json").read_text())
+        result["nativeStatus"] = native["status"]
+        result["nativeEvidenceSha256"] = {path.name: sha(path) for path in sorted(native_dir.iterdir()) if path.is_file()}
+        return native_dir, native
+
+    try:
+        if args.baseline:
+            assert result["apkSha256"] == M10_APK_SHA256, "Baseline requires the exact verified M10 app"
+        with socket.socket() as probe:
+            assert probe.connect_ex(("127.0.0.1", 18081)) != 0, "Fixture port18081 already owned"
+        with (evidence / "fixture.log").open("wb") as log:
+            fixture = subprocess.Popen([args.node, str(root / "fixture/server.cjs")], stdout=log, stderr=subprocess.STDOUT)
+        result["fixturePid"] = fixture.pid
+        deadline = time.monotonic() + 5
+        while True:
+            assert fixture.poll() is None, "Owned fixture exited during startup"
+            try:
+                remote("/__state")
+                break
+            except Exception:
+                if time.monotonic() >= deadline:
+                    raise
+                time.sleep(0.1)
+        before = remote("/__m14-reset", {})
+        assert before["items"] == before["requests"] == []
+        assert before["pressure"] == {"delayMs": 500, "inFlightMutations": 0, "peakInFlightMutations": 0, "requests": []}
+        original_network = network("initial")
+        result["networkBefore"] = original_network
+        assert original_network == ONLINE
+        for label, apk in (("app", args.apk), ("test", args.test_apk)):
+            assert "Success" in adb("install-" + label, "install", "-r", str(Path(apk).resolve()))
+        adb("stop-before-setup", "shell", "am", "force-stop", PACKAGE)
+        assert adb("clear-before-setup", "shell", "pm", "clear", PACKAGE) == "Success"
+        quiet_reset()
+        result["networkAtNativeStart"] = network("setup-offline", OFFLINE)
+        adb("wake", "shell", "input", "keyevent", "KEYCODE_WAKEUP")
+        adb("keyguard", "shell", "wm", "dismiss-keyguard")
+        adb("clear-logcat", "logcat", "-c")
+        try:
+            suite = adb("instrumentation", "shell", "am", "instrument", "-w", "-r", "-e", "class", TEST,
+                        PACKAGE + ".test/androidx.test.runner.AndroidJUnitRunner", timeout=360)
+        finally:
+            # Preserve partial native proof even when instrumentation failed or timed out.
+            native_dir, native = collect_native()
+        result["instrumentationCodes"] = re.findall(r"^INSTRUMENTATION_STATUS_CODE: (-?\d+)$", suite, re.M)
+        state, mutations = remote("/__m14-state"), remote("/__mutation-state")
+        result.update(remote=state, mutations=mutations)
+        assert native["status"] == "PASS", native.get("failure", native)
+        assert "OK (1 test)" in suite and "FAILURES" not in suite
+        assert result["instrumentationCodes"] == ["1", "0"]
+        result["fixtureAudit"] = audit_fixture(state, mutations, inputs)
+        result["nativeAudit"] = audit_native(native_dir, native, inputs, result["fixtureAudit"])
+        result["status"] = "PASS_BASELINE_ALREADY_CORRECT" if args.baseline else "PASS"
+    except Exception as error:
+        result.update(status="FAIL", error=repr(error))
+    finally:
+        try:
+            adb("logcat", "logcat", "-d", "-v", "threadtime")
+            if fixture is not None and fixture.poll() is None:
+                result["remoteBeforeCleanup"] = remote("/__m14-state")
+                result["mutationsBeforeCleanup"] = remote("/__mutation-state")
+            adb("cleanup-stop", "shell", "am", "force-stop", PACKAGE)
+            result["pidAfterCleanup"] = adb("cleanup-pid", "shell", "pidof", PACKAGE, check=False)
+            assert commands[-1]["exit"] in (0, 1) and not result["pidAfterCleanup"]
+            if original_network is not None:
+                result["networkAfter"] = network("cleanup-online", original_network)
+        except Exception as error:
+            result.update(status="FAIL", cleanupError=repr(error))
+        if fixture is not None:
+            fixture.terminate()
+            try:
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                assert result["fixtureExit"] == 0
+            except Exception as error:
+                fixture.kill()
+                result["fixtureExit"] = fixture.wait(timeout=5)
+                result.update(status="FAIL", fixtureCleanupError=repr(error))
+        with socket.socket() as probe:
+            result["fixturePortFree"] = probe.connect_ex(("127.0.0.1", 18081)) != 0
+        if fixture is not None and not result["fixturePortFree"]:
+            result.update(status="FAIL", fixtureCleanupError="Port18081 still has a listener")
+        result.update(elapsedSeconds=time.monotonic() - started, adbCommands=len(commands))
+        save()
+        print(json.dumps(result), flush=True)
+    return 1 if result["status"] == "FAIL" else 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())
diff --git a/src/App.tsx b/src/App.tsx
index 17f5a4c..99e74cd 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -2,8 +2,8 @@ import React, {useEffect, useRef, useState} from 'react';
 import {Button, DeviceEventEmitter, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
 import {Item} from './items';
 import {ItemMutation, ItemStore, openItemStore, openRuntimeItemStore} from './itemStore';
-import {ForegroundSync, SyncSession} from './sync';
-import {backgroundBridge, BackgroundState, ownsAutomaticSync, schedulePending, serializeSync} from './backgroundSync';
+import {assertSyncActive, ForegroundSync, SyncSession} from './sync';
+import {backgroundBridge, BackgroundState, observeForegroundSync, ownsAutomaticSync, schedulePending, serializeSync} from './backgroundSync';
 
 const defaultSync = (store: ItemStore, identityPrefix?: string, testRefreshClock = false) => {
   // Android supplies this prop only behind BuildConfig.DEBUG. Real network and
@@ -44,6 +44,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
+  const foreground = useRef<AbortController | null>(null);
   const busyRef = useRef(true);
   const mounted = useRef(false);
   const [{draft, editingId}, setEditor] = useState(() => editorMemory.current);
@@ -68,7 +69,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         if (mounted.current) {setBackgroundError('Background state unavailable; saved changes are retained.');}
       });
     });
-    return () => {mounted.current = false; subscription.remove();};
+    return () => {mounted.current = false; foreground.current?.abort(); subscription.remove();};
   }, []);
 
   function updateEditor(next: EditorState) {
@@ -185,22 +186,36 @@ export default function App({openStore = openItemStore, createSync = defaultSync
 
   async function synchronize(manual: boolean) {
     if (!mounted.current || !store.current || !sync.current || busyRef.current) {return;}
+    const origin = store.current;
+    const session = sync.current;
+    const current = () => mounted.current && store.current === origin && sync.current === session;
     busyRef.current = true;
     setBusy(true);
     setRefresh({status: 'refreshing'});
+    const observation = observeForegroundSync();
+    const controller = observation.controller;
+    foreground.current = controller;
     try {
+      await observation.ready;
+      assertSyncActive(controller.signal);
       const performed = await serializeSync(async () => {
-        if (!mounted.current) {return false;}
-        if (manual) {setBackground(await backgroundBridge.prepareManual());}
+        assertSyncActive(controller.signal);
+        if (!current()) {return false;}
+        if (manual) {
+          const state = await backgroundBridge.prepareManual();
+          if (current()) {setBackground(state);}
+        }
         else if (ownsAutomaticSync(await backgroundBridge.getState())) {return false;}
-        await sync.current!.synchronize();
+        assertSyncActive(controller.signal);
+        await session.synchronize(controller.signal);
+        assertSyncActive(controller.signal);
         return true;
       });
-      if (!performed) {if (mounted.current) {setRefresh({status: 'stale'});} return;}
-      if (!mounted.current) {return;}
-      const saved = await store.current.read();
-      const lastRefresh = await store.current.readLastSuccessfulRefresh();
-      if (mounted.current) {
+      if (!performed) {if (current()) {setRefresh({status: 'stale'});} return;}
+      if (!current()) {return;}
+      const saved = await origin.read();
+      const lastRefresh = await origin.readLastSuccessfulRefresh();
+      if (current()) {
         setItems(saved);
         setLastSuccessfulRefreshAt(lastRefresh);
         setRefresh({status: 'fresh'});
@@ -208,24 +223,28 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     } catch (reason) {
       // A conflict can commit its canonical winner before a later GET fails.
       // Show that committed state, while retaining the refresh error/time.
-      if (mounted.current) {
+      if (current()) {
         try {
-          const saved = await store.current.read();
-          if (mounted.current) {setItems(saved);}
+          const saved = await origin.read();
+          if (current()) {setItems(saved);}
         } catch { /* Keep the last confirmed list. */ }
-        if (mounted.current) {setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});}
+        if (current()) {setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});}
       }
     } finally {
       try {
-        const state = await serializeSync(() => mounted.current
-          ? schedulePending(store.current!) : backgroundBridge.getState());
-        if (mounted.current) {setBackground(state);}
+        await observation.dispose();
       } catch {
-        if (mounted.current) {setBackgroundError('Background scheduling failed; saved changes are retained.');}
+        if (current()) {setBackgroundError('Synchronization cleanup failed; saved changes are retained.');}
       }
-      await reloadPending();
-      busyRef.current = false;
-      if (mounted.current) {setBusy(false);}
+      try {
+        const state = await serializeSync(() => current()
+          ? schedulePending(origin) : backgroundBridge.getState());
+        if (current()) {setBackground(state);}
+      } catch {
+        if (current()) {setBackgroundError('Background scheduling failed; saved changes are retained.');}
+      }
+      if (current()) {await reloadPending(); busyRef.current = false; setBusy(false);}
+      if (foreground.current === controller) {foreground.current = null;}
     }
   }
 
diff --git a/src/backgroundSync.ts b/src/backgroundSync.ts
index 1a29104..13d5088 100644
--- a/src/backgroundSync.ts
+++ b/src/backgroundSync.ts
@@ -12,7 +12,39 @@ export interface BackgroundBridge {
   requestFinished(token: string): Promise<void>;
   complete(token: string, outcome: 'success' | 'retry' | 'failure'): Promise<boolean>;
 }
-export const backgroundBridge: BackgroundBridge = NativeModules.BackgroundSync;
+interface ForegroundNetworkBridge {
+  observeForegroundNetwork(token: string): Promise<boolean>;
+  stopObservingForegroundNetwork(token: string): Promise<void>;
+}
+export const backgroundBridge: BackgroundBridge & ForegroundNetworkBridge = NativeModules.BackgroundSync;
+
+let nextForeground = 0;
+export function observeForegroundSync(bridge: ForegroundNetworkBridge = backgroundBridge) {
+  const token = `foreground-${++nextForeground}`;
+  const controller = new AbortController();
+  let disposed = false;
+  let cleanup: Promise<void> | undefined;
+  // Install ownership before native registration or the shared upload queue.
+  // A delayed event from an older operation cannot cancel a reconnect.
+  const listener = DeviceEventEmitter.addListener('ForegroundSyncOffline', (event: {token: string}) => {
+    if (!disposed && event.token === token) {controller.abort();}
+  });
+  const ready = Promise.resolve().then(() => bridge.observeForegroundNetwork(token)).then(connected => {
+    if (!connected) {controller.abort();}
+  });
+  return {controller, ready, dispose: () => {
+    if (!cleanup) {
+      disposed = true;
+      listener.remove();
+      controller.abort();
+      // A bridge rejection may follow a native registration. Exact-token stop
+      // is safe in both cases, including cleanup before registration resolves.
+      cleanup = ready.then(() => bridge.stopObservingForegroundNetwork(token),
+        () => bridge.stopObservingForegroundNetwork(token));
+    }
+    return cleanup;
+  }};
+}
 
 // One JS runtime serves both the Activity and WorkManager. Keep the whole
 // upload/ACK operation serialized; scheduling waits only for durable enqueue.
@@ -25,12 +57,12 @@ export function serializeSync<T>(operation: () => Promise<T>): Promise<T> {
 
 export const ownsAutomaticSync = (state: BackgroundState) => state.status === 'active' || state.status === 'deferred';
 
-export async function schedulePending(store: ItemStore, bridge = backgroundBridge): Promise<BackgroundState> {
+export async function schedulePending(store: ItemStore, bridge: BackgroundBridge = backgroundBridge): Promise<BackgroundState> {
   const pending = await store.readPending();
   return pending[0]?.terminalError === null ? bridge.schedule() : bridge.getState();
 }
 
-export async function runBackgroundTask(task: {token: string}, bridge = backgroundBridge,
+export async function runBackgroundTask(task: {token: string}, bridge: BackgroundBridge = backgroundBridge,
   openStore: () => Promise<ItemStore> = openRuntimeItemStore,
   request: JsonRequest = (url, options) => fetch(url, options)): Promise<void> {
   const controller = new AbortController();
@@ -69,7 +101,7 @@ export async function runBackgroundTask(task: {token: string}, bridge = backgrou
             if (reservationMayExist) {await bridge.requestFinished(task.token);}
           }
         };
-        await new ForegroundSync(store, undefined, guardedRequest).uploadPending();
+        await new ForegroundSync(store, undefined, guardedRequest).uploadPending(controller.signal);
         outcome = 'success'; // Only after the existing native ACK transaction.
       } catch {
         if (!controller.signal.aborted) {
diff --git a/src/sync.ts b/src/sync.ts
index f82cea8..e8deae9 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -11,7 +11,11 @@ export type JsonRequest = (url: string, options: {
 export interface SyncSession {
   readonly initialized: boolean;
   readonly identityPrefix: string;
-  synchronize(): Promise<void>;
+  synchronize(signal?: AbortSignal): Promise<void>;
+}
+
+export function assertSyncActive(signal?: AbortSignal) {
+  if (signal?.aborted) {throw new Error('Synchronization cancelled');}
 }
 
 function remoteItem(value: unknown): Item {
@@ -52,14 +56,19 @@ export class ForegroundSync implements SyncSession {
 
   get initialized() {return this.refreshed;}
 
-  private async exchange(method: string, path: string, expectedStatus: number, body?: unknown, operation?: PendingMutation) {
+  private async exchange(method: string, path: string, expectedStatus: number, body?: unknown,
+    operation?: PendingMutation, signal?: AbortSignal) {
+    assertSyncActive(signal);
     const response = await this.request(`${this.baseUrl}${path}`, {
       method, headers: {'Content-Type': 'application/json'},
       ...(body === undefined ? {} : {body: JSON.stringify(body)}),
+      ...(signal === undefined ? {} : {signal}),
     });
+    assertSyncActive(signal);
     if (response.status !== expectedStatus) {
       if (operation && response.status === 409) {
         const failure = await response.json() as {error?: string; item?: unknown; tombstone?: unknown};
+        assertSyncActive(signal);
         if (failure?.error === 'identity_conflict') {
           await this.store.rejectIdentity(operation);
           throw new Error('Mutation identity conflict; upload stopped without retry');
@@ -73,13 +82,15 @@ export class ForegroundSync implements SyncSession {
       }
       throw new Error(`${method} ${path} failed (HTTP ${response.status})`);
     }
-    return response.json();
+    const result = await response.json();
+    assertSyncActive(signal);
+    return result;
   }
 
-  async synchronize(): Promise<void> {
-    await this.uploadPending();
+  async synchronize(signal?: AbortSignal): Promise<void> {
+    await this.uploadPending(signal);
 
-    const response = await this.exchange('GET', '/items', 200) as {items?: unknown; tombstones?: unknown};
+    const response = await this.exchange('GET', '/items', 200, undefined, undefined, signal) as {items?: unknown; tombstones?: unknown};
     if (!Array.isArray(response.items) || (response.tombstones !== undefined && !Array.isArray(response.tombstones))) {
       throw new Error('Invalid remote snapshot');
     }
@@ -88,22 +99,25 @@ export class ForegroundSync implements SyncSession {
     if (new Set([...snapshot, ...tombstones].map(item => item.id)).size !== snapshot.length + tombstones.length) {
       throw new Error('Duplicate Item or tombstone in remote snapshot');
     }
+    assertSyncActive(signal);
     await this.store.replaceSnapshot(snapshot, this.now(), tombstones);
     this.refreshed = true;
   }
 
-  async uploadPending(): Promise<void> {
+  async uploadPending(signal?: AbortSignal): Promise<void> {
     // Recover ordered upload intent from SQLite, including on the first sync
     // after process restart. Each edit is sent separately, without coalescing.
     for (;;) {
+      assertSyncActive(signal);
       const operation = await this.store.prepareNext();
+      assertSyncActive(signal);
       if (operation === null) {break;}
       if (operation.terminalError === 'identity_conflict') {
         throw new Error('Mutation identity conflict; upload stopped without retry');
       }
       const target = mutationTarget(operation.kind, operation.itemId);
       const response = await this.exchange(target.method, target.path, target.status,
-        {...operation.payload, clientMutationId: operation.clientMutationId, payloadHash: operation.payloadHash}, operation) as {item?: unknown; deletedId?: unknown} | null;
+        {...operation.payload, clientMutationId: operation.clientMutationId, payloadHash: operation.payloadHash}, operation, signal) as {item?: unknown; deletedId?: unknown} | null;
       if (response === null) {continue;} // Preserved conflict is no longer pending.
       let result: MutationResult;
       if (operation.kind === 'delete') {
@@ -114,6 +128,9 @@ export class ForegroundSync implements SyncSession {
         if (item.id !== operation.itemId) {throw new Error('Remote acknowledgment belongs to another Item');}
         result = {item};
       }
+      // Cancellation does not roll back an ACK transaction already entered.
+      // A cancelled/late response must never enter that transaction instead.
+      assertSyncActive(signal);
       await this.store.acknowledge(operation, result);
     }
   }
diff --git a/verification/M14-inputs.json b/verification/M14-inputs.json
new file mode 100644
index 0000000..ed70412
--- /dev/null
+++ b/verification/M14-inputs.json
@@ -0,0 +1,149 @@
+{
+  "profile": "phase-1",
+  "thread": "M14",
+  "specRevision": "61280dd86ce88b6e431f408241c0998a275960aa",
+  "start": "568d620c2d25af44764b8b591d89bc32c0b786f8",
+  "identityPrefix": "pressure",
+  "items": [
+    {
+      "id": "pressure-001",
+      "title": "Pressure 001",
+      "completed": false
+    },
+    {
+      "id": "pressure-002",
+      "title": "Pressure 002",
+      "completed": false
+    },
+    {
+      "id": "pressure-003",
+      "title": "Pressure 003",
+      "completed": false
+    },
+    {
+      "id": "pressure-004",
+      "title": "Pressure 004",
+      "completed": false
+    },
+    {
+      "id": "pressure-005",
+      "title": "Pressure 005",
+      "completed": false
+    },
+    {
+      "id": "pressure-006",
+      "title": "Pressure 006",
+      "completed": false
+    },
+    {
+      "id": "pressure-007",
+      "title": "Pressure 007",
+      "completed": false
+    },
+    {
+      "id": "pressure-008",
+      "title": "Pressure 008",
+      "completed": false
+    },
+    {
+      "id": "pressure-009",
+      "title": "Pressure 009",
+      "completed": false
+    },
+    {
+      "id": "pressure-010",
+      "title": "Pressure 010",
+      "completed": false
+    },
+    {
+      "id": "pressure-011",
+      "title": "Pressure 011",
+      "completed": false
+    },
+    {
+      "id": "pressure-012",
+      "title": "Pressure 012",
+      "completed": false
+    }
+  ],
+  "envelopes": [
+    {
+      "itemId": "pressure-001",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-001\",\"title\":\"Pressure 001\"}}",
+      "payloadHash": "ae07b5d93d038ef1068755451d9cf6ed6fd415234fc834bb01e4b1698e2a713f"
+    },
+    {
+      "itemId": "pressure-002",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-002\",\"title\":\"Pressure 002\"}}",
+      "payloadHash": "29e613dfa77b8e6910680f063fa0c904cb972a580c83d6fce4ab353bcf19e9b5"
+    },
+    {
+      "itemId": "pressure-003",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-003\",\"title\":\"Pressure 003\"}}",
+      "payloadHash": "26b52518fe6f7145c3702c0eeb1c254888cc33cb0640bfd899ccd658d22cbdac"
+    },
+    {
+      "itemId": "pressure-004",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-004\",\"title\":\"Pressure 004\"}}",
+      "payloadHash": "6ae7172640b9c23da3c0ff578611b1243a1f6b9ad688bf45104be7db5340def5"
+    },
+    {
+      "itemId": "pressure-005",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-005\",\"title\":\"Pressure 005\"}}",
+      "payloadHash": "98c1b632b8da7811e6df23fc13e2bd685760d00bb84cbe74e067339952825285"
+    },
+    {
+      "itemId": "pressure-006",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-006\",\"title\":\"Pressure 006\"}}",
+      "payloadHash": "7b9ec807cd9363b8935959287284a4f18ae08dbeac85dc775351ec9d39793a25"
+    },
+    {
+      "itemId": "pressure-007",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-007\",\"title\":\"Pressure 007\"}}",
+      "payloadHash": "3013081e7f0959ad4c2b60f1aeab7df2110a594176f91d9f5eb0869e22b439d4"
+    },
+    {
+      "itemId": "pressure-008",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-008\",\"title\":\"Pressure 008\"}}",
+      "payloadHash": "675bf54598ee25e8f61be10926a97d607fa94752b8762bfab7a9759608bf7f9f"
+    },
+    {
+      "itemId": "pressure-009",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-009\",\"title\":\"Pressure 009\"}}",
+      "payloadHash": "d452b2dd73c5040e79d46ca57c1c869c5dd60204619931dba0d33bf4a40c13d0"
+    },
+    {
+      "itemId": "pressure-010",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-010\",\"title\":\"Pressure 010\"}}",
+      "payloadHash": "6be83218520a3faccb5e28ecb7503aec56bdfbc4dacc631686dc22665dba1861"
+    },
+    {
+      "itemId": "pressure-011",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-011\",\"title\":\"Pressure 011\"}}",
+      "payloadHash": "861951569c5960b3d77c6986561778b6398deb36d6afa5e8e138a153465bbd6c"
+    },
+    {
+      "itemId": "pressure-012",
+      "canonical": "{\"method\":\"POST\",\"path\":\"/items\",\"payload\":{\"completed\":false,\"id\":\"pressure-012\",\"title\":\"Pressure 012\"}}",
+      "payloadHash": "f85243c5292cc488ec16889c3fe25a4c58fcc51564e47ec21e0fedf120245948"
+    }
+  ],
+  "nativeScheduleRequests": 4,
+  "foregroundBurstRequests": 1,
+  "foregroundReconnectRequests": 1,
+  "barrierAcknowledgments": 3,
+  "responseDelayMs": 500,
+  "maxConcurrentMutations": 2,
+  "firstTimestamp": 1700001000000,
+  "finalNextTimestamp": 1700001012000,
+  "uiTimeoutMs": 15000,
+  "coordinatorTimeoutMs": 5000,
+  "networkTimeoutMs": 10000,
+  "settlementTimeoutMs": 10000,
+  "drainTimeoutMs": 30000,
+  "fixturePort": 18081,
+  "setup": "Existing React testIdentityPrefix prop only. All12 creates use production UI/SQLite; generated distinct mutation identities captured before dispatch. No queue or WorkDatabase injection.",
+  "barrier": "One consistent native SQLite read proves exactly3 durableACK before offline/cancellation; server receipt count alone is not accepted.",
+  "schedulerAttribution": "One preattached instrumentation session, actual nonforced null-namespace JobScheduler callback, Greedy enabled. No new process-death claim.",
+  "burstAccounting": "4 native schedule calls +1 accepted ordinary foreground Sync during active initial drain =5; reconnect foreground Sync is separate."
+}
diff --git a/verification/M14.md b/verification/M14.md
new file mode 100644
index 0000000..332ac5e
--- /dev/null
+++ b/verification/M14.md
@@ -0,0 +1,82 @@
+# M14 — bounded pressure and cancellation (phase-1)
+
+**Main runtime acceptance: PASS.** Branch `track/react-native`; START
+`568d620c2d25af44764b8b591d89bc32c0b786f8`; SPEC_REVISION
+`61280dd86ce88b6e431f408241c0998a275960aa`.
+Attempt10 / repair9 / fixed12 invocations9: eight failed runtime invocations and
+the separate repair5 compilation failure remain preserved. Continued repairs use
+the [latest user authorization](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/REPAIR-AUTHORIZATION-2026-08-29.md),
+without resetting counts or changing the [fixed scenario](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/profiles/phase-1/M14-scenario.md)
+or [approved observation plan](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/PREFLIGHT-M14.md).
+
+## Baseline and preserved failures
+
+The original verified M10 [APK](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M10/m10-candidate.apk)
+(`7f8c1110bc38ca195d1572d6b419a9e5a3dc97cb5441df208aa70900fe8b5c27`)
+was unchanged through invocation6. The first five runs exposed test preparation
+defects, not product limitations. After those repairs, [invocation6](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M14/main-repair6-failure-01.json)
+reached three durable ACKs/nine pending intents but foreground transport failed
+to settle after offline cancellation. Product edits began only after that audit.
+Repair7 fixed cancellation; runs7–8 still failed reconnect. The passive run8
+diagnostic confirmed reuse of an idle pre-loss socket and its original EOF.
+
+The [final immutable manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/repair9-candidate-manifest.json)
+(`e72deb799de383b4eee52877f7241c17d19c17c33b9bb610a2a40b5ac127de24`)
+links every prior candidate, APK, raw failure, repair dispatch, compiler failure,
+source snapshot and check. Its pre-execution status is historical. No failure or
+old artifact was overwritten, and no owner device warmup was run.
+
+## Frozen implementation and reproduction
+
+The existing upload loop now receives foreground cancellation before queue entry
+and rejects cancelled/late responses before durable result commits. Native loss
+ownership/listener cleanup is independent of Worker tokens. Repair9 evicts idle
+connections from the actual factory-owned pool on owned loss and loss-marked
+cleanup. It does not retry requests or change WorkManager's allowance, schema,
+identity, conflict policy or fixture timing. Active-to-idle race closure is not
+claimed beyond the measured dispatcher/UI settlement.
+
+The manifest pins all72 execution files, source/build snapshots, wrapper and
+`implementationArgv` (without `--baseline`), including the frozen native test,
+inputs and external harness. Final [app APK](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/m14-repair9-app.apk)
+SHA256 is `a1525b587f5566d7c889a9244a50a28811787c1fdbd1d4e8586e316f5d3ce088`;
+the reused [repair8 test APK](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/m14-repair8-test.apk)
+is `dbba019f6aae2d49631c2234b070b052e0a6d794bf06039bf05a6bb87ef871d4`.
+Only this ledger and TRACK change after runtime acceptance.
+
+## Verification
+
+- Focused [host checks](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/repair7-host-01.command.json)
+  passed67, skipped2, including13 new cancellation/ownership cases;
+  [typecheck](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/repair7-typecheck-01.command.json)
+  passed. Those unchanged JS/test results were reused for repair9. The two skipped
+  fixed M10 fixture cases were deliberately not replayed; no host fixed12 run occurred.
+- Repair9 [app build and existing JVM checks](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/repair9-native-build-01.command.json)
+  passed (17.809s;2 JVM tests). Main independently checked native/bundle/artifact
+  bytes in its [preflight](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M14/main-repair9-preflight.json).
+  Earlier M05–M10 Android evidence is retained, not rerun; no exhausted M10-A
+  invocation was reused as a regression or hidden warmup.
+- Main's [final runtime audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M14/main-repair9-acceptance.json)
+  (SHA256 `5ce683bebbee890c16d69ec75f6e625de853b0c8ab12a242c4d7bd81352d8cc0`)
+  independently bound172 raw files, six native DB checkpoints,72 sources and both
+  APKs. Its manifest hash refers to [main's normalized execution manifest](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M14/main-repair9-tested-manifest.json),
+  which separately pins the owner manifest above.
+  [Actual run](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M14/main-repair9-android-m14-01/result.json):
+  PASS53.780s/44ADB, same PID18615/Activity/ReactContext; twelve real UI creates;
+  four production native scheduling requests plus one accepted foreground burst
+  request; actual registered WorkManager/SystemJobService activity with Greedy enabled.
+- The native barrier observed exactly3 durable ACKs/9 original pending intents
+  before actual offline cancellation. Transport/UI settled in516ms. One foreground
+  reconnect request completed convergence, while normal background work started
+  the first fresh post-restore request **before** the click; not all restored
+  requests are attributed to foreground. Final state:12 exact canonical v1 Items,
+  pending0, applied12, duplicate0, nextTimestamp1700001012000. Actual peak mutation
+  HTTP1 (limit2), with real500ms per-request delay and no fixture serialization lock.
+- Actual pool193396921 idle1→0 on owned loss; all post-restore connections were
+  fresh. All13 calls retained the original transport policy, including retry=false;
+  no IOException occurred, and the passive test seam was restored exactly.
+  [Main cleanup](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M14/main-repair9-cleanup-01/result.json)
+  verified fixture63275 exit0/absent, app absent, ports18080/18081 free and network0/1/1.
+
+No remaining M14 runtime acceptance failure. Final Git/tag audit belongs to main;
+M15 remains unstarted. No phase-2 feature or push is included.
