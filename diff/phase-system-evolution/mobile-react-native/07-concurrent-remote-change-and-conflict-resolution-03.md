## `feat(M07): preserve stale intent with versioned canonical reconciliation`

diff --git a/TRACK.md b/TRACK.md
index 348c8bf..04618e6 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -139,6 +139,36 @@ backoff, push, state framework or extra business feature is introduced. The froz
 M06 Android harness covers lost response, actual process death/replay, a separate
 production-created collision and wrong-hash rejection. See `verification/M06.md`.
 
+## M07: stale versions and preserved conflicts (phase-1)
+
+Schema v5 separates observed canonical versions/Item payloads from optimistic
+local Items. Updates and deletes carry a hashed `baseVersion`. A dispatch marker
+commits before sending; once set, the identity, payload and hash cannot change.
+An acknowledged own predecessor may prepare only its next never-dispatched
+same-Item successor, in the acknowledgment/dequeue transaction. This preserves
+M05's four separate accepts without rebasing from an external GET or conflict.
+
+The fixture rejects stale bases with the canonical Item or versioned tombstone.
+One local transaction preserves the original intent and conflict response,
+removes it from ordinary drains, and accepts the canonical winner. The screen
+explains that original attempts remain saved and will not retry. A fresh explicit
+edit uses the latest observed version and a new identity; old evidence remains.
+There is no merge editor, automatic retry, scheduler or new dependency.
+
+Observed acknowledgment payloads remain distinct from optimistic fields, so a
+late older snapshot cannot publish optimistic data as canonical. DELETE receipts
+record deletion at the accepted base+1, with unknown timestamp until a real
+tombstone arrives. No timestamp is fabricated. Exact identity replay is checked
+before current-version rejection and still returns its original result.
+
+Schema4 cached Item provenance is unknown even with an empty queue (an ACK may
+have preceded a failed GET). Migration keeps the cache readable without guessing
+canonical versions. Possibly-sent unversioned updates/deletes retain their exact
+identity/payload/hash as blocked conflict evidence; legacy creates and terminal
+identity errors retain their existing semantics. No migration framework is added.
+See `verification/M07.md` for the preserved baseline failure/repair, host checks,
+and main's official Android result.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 492de4d..ef20c47 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -245,10 +245,11 @@ test('M05 UI reads durable pending work after restart and clears it after a fore
   // execute the real fixture. Native-bridge host SQLite transactions remain real.
   const replies = [
     {method: 'POST', status: 201, body: {id: 'device-001', title: 'Gamma', completed: false}, response: {item: gamma}},
-    {method: 'PATCH', status: 200, body: {title: 'Queued edit'}, response: {item: renamed}},
-    {method: 'PATCH', status: 200, body: {completed: true}, response: {item: completed}},
-    {method: 'DELETE', status: 200, body: null, response: {deletedId: 'remote-002'}},
-    {method: 'GET', status: 200, body: null, response: {items: final}},
+    {method: 'PATCH', status: 200, body: {title: 'Queued edit', baseVersion: 1}, response: {item: renamed}},
+    {method: 'PATCH', status: 200, body: {completed: true, baseVersion: 2}, response: {item: completed}},
+    {method: 'DELETE', status: 200, body: {baseVersion: 1}, response: {deletedId: 'remote-002'}},
+    {method: 'GET', status: 200, body: null, response: {items: final,
+      tombstones: [{id: 'remote-002', version: 2, updatedAt: 1700000303000, deleted: true}]}},
   ];
   let offline = true;
   const request: JsonRequest = jest.fn(async (_url, options) => {
@@ -338,3 +339,57 @@ test('M06 startup exposes a persisted identity collision and an explicit drain c
   expect(request).not.toHaveBeenCalled();
   expect((await reopened.readPending())[0].terminalError).toBe('identity_conflict');
 });
+
+test('M07 visible canonical-wins notice survives reopening and a fresh explicit edit keeps the old attempt', async () => {
+  const input = require('../verification/M07-inputs.json');
+  const scenario = input.cases[0];
+  const replies = [
+    {method: 'PATCH', status: 409, body: scenario.envelope.wireBody, response: scenario.conflictResponse},
+    {method: 'GET', status: 200, body: null, response: {items: [input.remoteWinner]}},
+    {method: 'PATCH', status: 200, body: input.explicitEnvelope.wireBody, response: {item: input.explicitItem}},
+    {method: 'GET', status: 200, body: null, response: {items: [input.explicitItem]}},
+  ];
+  const request: JsonRequest = jest.fn(async (_url, options) => {
+    const reply = replies.shift()!;
+    expect(options.method).toBe(reply.method);
+    expect(options.body ? JSON.parse(options.body) : null).toEqual(reply.body);
+    return {status: reply.status, json: async () => reply.response};
+  });
+  const store = await openItemStore(undefined, () => scenario.clientMutationId);
+  await store.replaceSnapshot([input.seed]);
+  const createSync = (opened: typeof store) => new ForegroundSync(opened, 'http://fixed-m07', request);
+  const view = render(<App openStore={async () => store} createSync={createSync} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Initial'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Local attempt');
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await saved();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(screen.getByText('Remote winner')).toBeTruthy();
+  expect(screen.queryByText('Local attempt')).toBeNull();
+  expect(screen.getByLabelText('Pending uploads: 0')).toBeTruthy();
+  expect(screen.getByLabelText('Conflicts preserved: 1')).toBeTruthy();
+  expect(screen.getByText(/Canonical state wins after refresh.*Original attempts are saved and will not retry/)).toBeTruthy();
+  view.unmount();
+  closeDatabases();
+  const reopened = await openItemStore(undefined, () => input.explicitEnvelope.clientMutationId);
+  render(<App openStore={async () => reopened} createSync={createSync} />);
+  await saved();
+  expect(request).toHaveBeenCalledTimes(2);
+  expect(screen.getByLabelText('Conflicts preserved: 1')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Edit Remote winner'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Explicit edit');
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await saved();
+  expect((await reopened.readPending())[0]).toMatchObject({clientMutationId: input.explicitEnvelope.clientMutationId,
+    payload: input.explicitEnvelope.payload, payloadHash: input.explicitEnvelope.payloadHash});
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(screen.getByText('Explicit edit')).toBeTruthy();
+  expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+  expect(screen.getByLabelText('Conflicts preserved: 1')).toBeTruthy();
+  expect(await reopened.readConflicts()).toEqual([scenario.conflictEvidence]);
+  expect(await reopened.read()).toEqual([input.explicitItem]);
+  expect(replies).toEqual([]);
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index 9ec5b8c..d89ab70 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -1,6 +1,6 @@
 import {Item, itemsReducer} from '../src/items';
 import SQLite from 'react-native-sqlite-2';
-import {ItemMutation, itemToRow, openItemStore, PendingMutation, rowToItem} from '../src/itemStore';
+import {ItemMutation, itemToRow, openItemStore, PendingMutation, rowToItem, SCHEMA_VERSION} from '../src/itemStore';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
 import {canonicalJson, mutationHash, mutationTarget} from '../src/mutationProtocol';
 
@@ -90,11 +90,11 @@ test.each([/^INSERT INTO items/, /^END/])('M02 failure at %s rolls back Item and
 test('M02 unsupported schema is rejected without recreating or deleting existing data', async () => {
   const store = await openItemStore();
   const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  connection().exec('PRAGMA user_version = 5');
+  connection().exec(`PRAGMA user_version = ${SCHEMA_VERSION + 1}`);
   closeDatabases();
-  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 5');
+  await expect(openItemStore()).rejects.toThrow(`Unsupported local database schema ${SCHEMA_VERSION + 1}`);
   expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(5);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(SCHEMA_VERSION + 1);
 });
 
 test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
@@ -133,7 +133,7 @@ test('M04 upgrades a literal M03 schema without touching cached Items or local i
   const store = await openItemStore(name);
   expect(await store.read()).toEqual(seeds);
   expect(await store.readLastSuccessfulRefresh()).toBeNull();
-  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(4);
+  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(5);
   expect(database.prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
   closeDatabases();
   const reopened = await openItemStore(name);
@@ -154,11 +154,11 @@ const m05Actions: ItemMutation[] = [
 ];
 const m05Pending = [
   {sequence: 1, kind: 'create', itemId: 'device-001', payload: {id: 'device-001', title: 'Gamma', completed: false}},
-  {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
-  {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
-  {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
+  {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit', baseVersion: 1}},
+  {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true, baseVersion: 1}},
+  {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: {baseVersion: 1}},
 ].map(operation => ({...operation, clientMutationId: expect.any(String),
-  payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null}));
+  payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null, dispatched: false}));
 
 const mutationFailures = [
   {index: 0, kind: 'create', localSql: /^INSERT INTO items/},
@@ -251,7 +251,7 @@ test('M05 v2 migration preserves Items, allocator and refresh time, including fa
   expect(await reopened.read()).toEqual(m05Seeds);
   expect(await reopened.readPending()).toEqual([]);
   expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(4);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(5);
 });
 
 test.each(m06.hashVectors)('M06 shared compact UTF-8 canonical vector $method $path $sha256', vector => {
@@ -266,7 +266,7 @@ test.each(m06.hashVectors)('M06 shared compact UTF-8 canonical vector $method $p
 });
 
 const m06Pending: PendingMutation = {sequence: 1, kind: 'create', itemId: 'crash-001', payload: m06.payload,
-  clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null};
+  clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null, dispatched: false};
 const lastAcknowledgement = () => {
   const value = connection().prepare('SELECT last_acknowledgement FROM sync_metadata WHERE singleton=1').get()?.last_acknowledgement;
   return value === null ? null : JSON.parse(String(value));
@@ -278,14 +278,16 @@ test.each([/^UPDATE sync_metadata SET last_acknowledgement/, /^DELETE FROM pendi
     const local = await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
     expect(await store.readPending()).toEqual([m06Pending]);
     expect(lastAcknowledgement()).toBeNull();
+    const sent = await store.prepareNext();
+    expect(sent).toEqual({...m06Pending, dispatched: true});
     failNextSql(failAt);
-    await expect(store.acknowledge(m06Pending, {item: m06.canonicalItem})).rejects.toThrow('injected persistence failure');
+    await expect(store.acknowledge(sent!, {item: m06.canonicalItem})).rejects.toThrow('injected persistence failure');
     closeDatabases();
     const reopened = await openItemStore();
-    expect(await reopened.readPending()).toEqual([m06Pending]);
+    expect(await reopened.readPending()).toEqual([sent]);
     expect(await reopened.read()).toEqual(local);
     expect(lastAcknowledgement()).toBeNull();
-    await reopened.acknowledge(m06Pending, {item: m06.canonicalItem});
+    await reopened.acknowledge(sent!, {item: m06.canonicalItem});
     expect(await reopened.readPending()).toEqual([]);
     expect(lastAcknowledgement()).toEqual(m06.acknowledgement);
     closeDatabases();
@@ -299,6 +301,7 @@ test('M06 foreign, out-of-order or terminal acknowledgments cannot erase intent'
   const store = await openItemStore(undefined, () => `check-${++next}`);
   await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
   await store.mutate({type: 'rename', id: 'crash-001', title: 'Still local', now: 1700000401000});
+  await store.prepareNext();
   const pending = await store.readPending();
   await expect(store.acknowledge(pending[0], {item: {...m06.canonicalItem, id: 'another'}})).rejects.toThrow('does not match');
   await expect(store.acknowledge({...pending[0], clientMutationId: 'another'}, {item: m06.canonicalItem})).rejects.toThrow('does not match');
@@ -316,12 +319,15 @@ test('M06 accepting an earlier intent records its result without overwriting a l
   const store = await openItemStore(undefined, () => `ordered-${++next}`);
   await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
   const edited = await store.mutate({type: 'rename', id: 'crash-001', title: 'Still local', now: 1700000401000});
+  await store.prepareNext();
   const pending = await store.readPending();
   await store.acknowledge(pending[0], {item: m06.canonicalItem});
   closeDatabases();
   const reopened = await openItemStore();
   expect(await reopened.read()).toEqual(edited);
-  expect(await reopened.readPending()).toEqual([pending[1]]);
+  const prepared = {title: 'Still local', baseVersion: 1};
+  expect(await reopened.readPending()).toEqual([{...pending[1], payload: prepared,
+    payloadHash: mutationHash('PATCH', '/items/crash-001', prepared)}]);
   expect(lastAcknowledgement()).toEqual({...m06.acknowledgement as object, clientMutationId: pending[0].clientMutationId});
 });
 
@@ -349,13 +355,13 @@ test.each([/^UPDATE pending_mutations SET client_mutation_id/, /^CREATE UNIQUE I
     expect(database.prepare('SELECT * FROM items').all()).toEqual(local);
     expect(database.prepare('SELECT * FROM sync_metadata').get()).toEqual({singleton: 1, last_successful_refresh_at: 1700000200000});
     const migrated = await openItemStore(undefined, () => m06.clientMutationId);
-    expect(await migrated.readPending()).toEqual([{...m06Pending, sequence: 7}]);
+    expect(await migrated.readPending()).toEqual([{...m06Pending, sequence: 7, dispatched: true}]);
     expect(await migrated.readLastSuccessfulRefresh()).toBe(1700000200000);
     expect(database.prepare('SELECT * FROM items').all()).toEqual(local);
     closeDatabases();
     const makeIdentity = jest.fn(() => 'new-after-upgrade');
     const reopened = await openItemStore(undefined, makeIdentity);
-    expect(await reopened.readPending()).toEqual([{...m06Pending, sequence: 7}]);
+    expect(await reopened.readPending()).toEqual([{...m06Pending, sequence: 7, dispatched: true}]);
     expect(makeIdentity).not.toHaveBeenCalled();
     await reopened.mutate({type: 'create', title: 'Next', now: 1700000401000}, 'crash');
     const next = (await reopened.readPending())[1];
@@ -363,3 +369,96 @@ test.each([/^UPDATE pending_mutations SET client_mutation_id/, /^CREATE UNIQUE I
     expect(next.itemId).toBe('crash-002');
     expect(next.clientMutationId).toBe('new-after-upgrade');
   });
+
+const m07 = require('../verification/M07-inputs.json');
+
+test.each([/^UPDATE pending_mutations SET payload =/, /^INSERT OR REPLACE INTO remote_versions/, /^END/])(
+  'M07 ACK failure at %s rolls back canonical observation and own-successor preparation with dequeue', async failAt => {
+    const input = m07.successorRegression;
+    const identities = [input.firstIdentity, input.successorIdentity];
+    const store = await openItemStore(undefined, () => identities.shift()!);
+    await store.replaceSnapshot([m07.seed]);
+    await store.mutate({type: 'rename', id: m07.seed.id, title: 'Own predecessor', now: m07.seed.updatedAt});
+    const optimistic = await store.mutate({type: 'toggle', id: m07.seed.id, now: m07.seed.updatedAt});
+    const first = (await store.prepareNext())!;
+    const pending = await store.readPending();
+    const known = connection().prepare('SELECT * FROM remote_versions').all();
+    failNextSql(failAt);
+    await expect(store.acknowledge(first, {item: input.firstResult})).rejects.toThrow('injected persistence failure');
+    closeDatabases();
+    const reopened = await openItemStore();
+    expect(await reopened.read()).toEqual(optimistic);
+    expect(await reopened.readPending()).toEqual(pending);
+    expect(lastAcknowledgement()).toBeNull();
+    expect(connection().prepare('SELECT * FROM remote_versions').all()).toEqual(known);
+    await reopened.acknowledge(first, {item: input.firstResult});
+    closeDatabases();
+    const recovered = await openItemStore();
+    expect(await recovered.read()).toEqual(optimistic);
+    expect(await recovered.readPending()).toEqual([{...pending[1], payload: input.preparedSuccessor.payload,
+      payloadHash: input.preparedSuccessor.payloadHash}]);
+    expect(lastAcknowledgement()).toEqual({clientMutationId: first.clientMutationId, payloadHash: first.payloadHash,
+      status: 200, result: {item: input.firstResult}});
+    console.info('M07_ATOMIC_OWN_PREPARATION', JSON.stringify({failAt: String(failAt), before: pending, after: await recovered.readPending()}));
+  });
+
+test.each([/^INSERT INTO mutation_conflicts/, /^DELETE FROM pending_mutations/, /^INSERT OR REPLACE INTO remote_versions/, /^INSERT INTO items/, /^END/])(
+  'M07 conflict failure at %s cannot split original evidence, canonical winner and dequeue', async failAt => {
+    const scenario = m07.cases[0];
+    const store = await openItemStore(undefined, () => scenario.clientMutationId);
+    await store.replaceSnapshot([m07.seed]);
+    const local = await store.mutate({type: 'rename', id: m07.seed.id, title: 'Local attempt', now: m07.seed.updatedAt});
+    const sent = (await store.prepareNext())!;
+    expect(sent).toEqual(scenario.conflictEvidence.intent);
+    const known = connection().prepare('SELECT * FROM remote_versions').all();
+    failNextSql(failAt);
+    await expect(store.rejectVersion(sent, m07.remoteWinner, null)).rejects.toThrow('injected persistence failure');
+    closeDatabases();
+    const reopened = await openItemStore();
+    expect(await reopened.read()).toEqual(local);
+    expect(await reopened.readPending()).toEqual([sent]);
+    expect(await reopened.readConflicts()).toEqual([]);
+    expect(connection().prepare('SELECT * FROM remote_versions').all()).toEqual(known);
+    await reopened.rejectVersion(sent, m07.remoteWinner, null);
+    closeDatabases();
+    const recovered = await openItemStore();
+    expect(await recovered.read()).toEqual([m07.remoteWinner]);
+    expect(await recovered.readPending()).toEqual([]);
+    expect(await recovered.readConflicts()).toEqual([scenario.conflictEvidence]);
+    expect(lastAcknowledgement()).toBeNull();
+    console.info('M07_ATOMIC_CONFLICT', JSON.stringify({failAt: String(failAt), conflict: await recovered.readConflicts()}));
+  });
+
+test('M07 an old snapshot uses observed ACK fields, never optimistic fields masquerading as canonical', async () => {
+  const input = m07.successorRegression;
+  const store = await openItemStore(undefined, () => input.firstIdentity);
+  await store.replaceSnapshot([m07.seed]);
+  const optimistic = await store.mutate({type: 'rename', id: m07.seed.id, title: 'Own predecessor', now: m07.seed.updatedAt});
+  await store.acknowledge((await store.prepareNext())!, {item: input.firstResult});
+  expect(await store.read()).toEqual(optimistic); // The receipt is separate until reconciliation.
+  expect(optimistic).not.toEqual([input.firstResult]);
+  closeDatabases();
+  const reopened = await openItemStore();
+  await reopened.replaceSnapshot([m07.seed]);
+  expect(await reopened.read()).toEqual([input.firstResult]);
+  expect(JSON.parse(String(connection().prepare('SELECT canonical_item FROM remote_versions').get()?.canonical_item)))
+    .toEqual(input.firstResult);
+});
+
+test('M07 a successful DELETE receipt prevents resurrection before its timestamped tombstone arrives', async () => {
+  const store = await openItemStore(undefined, () => m07.cases[1].clientMutationId);
+  await store.replaceSnapshot([m07.seed]);
+  await store.mutate({type: 'delete', id: m07.seed.id});
+  await store.acknowledge((await store.prepareNext())!, {deletedId: m07.seed.id});
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(connection().prepare('SELECT * FROM remote_versions').get()).toEqual({id: m07.seed.id, version: 2,
+    updated_at: null, deleted: 1, canonical_item: null});
+  await reopened.replaceSnapshot([m07.seed]);
+  expect(await reopened.read()).toEqual([]);
+  await reopened.replaceSnapshot([], undefined, [m07.remoteTombstone]);
+  expect(connection().prepare('SELECT * FROM remote_versions').get()).toEqual({id: m07.seed.id, version: 2,
+    updated_at: m07.remoteTombstone.updatedAt, deleted: 1, canonical_item: null});
+  await reopened.replaceSnapshot([m07.seed]);
+  expect(await reopened.read()).toEqual([]);
+});
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 366ad83..0874176 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -1,12 +1,14 @@
 /// <reference types="node" />
 import {request as httpRequest, Server} from 'node:http';
 import {ForegroundSync, JsonRequest} from '../src/sync';
-import {openItemStore} from '../src/itemStore';
+import {MutationConflict, openItemStore, PendingMutation} from '../src/itemStore';
+import {mutationHash, mutationTarget} from '../src/mutationProtocol';
+import {Item} from '../src/items';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
 
 type Trace = {method: string; path: string; body: unknown; status: number; response: unknown};
 const {createFixture} = require('../fixture/server.cjs') as {createFixture(): {
-  server: Server; reset(): void; state(): {items: unknown[]; nextTimestamp: number; requests: Trace[]};
+  server: Server; reset(): void; state(): {items: unknown[]; tombstones?: unknown[]; nextTimestamp: number; requests: Trace[]};
   mutationState(): {appliedCount: number; duplicateCount: number; conflictCount: number; hashRejectedCount: number; attempts: unknown[]};
 }};
 const fixture = createFixture();
@@ -46,6 +48,7 @@ const gamma = {id: 'device-001', title: 'Gamma', completed: false, version: 1, u
 const renamed = {...seeds[0], title: 'Alpha synced', version: 2, updatedAt: 1700000101000};
 const completed = {...renamed, completed: true, version: 3, updatedAt: 1700000102000};
 const final = [gamma, completed];
+const deleted = {id: 'remote-002', version: 2, updatedAt: 1700000103000, deleted: true};
 
 test('M03 frozen HTTP contract synchronizes two independent persistent databases exactly', async () => {
   const first = await openItemStore('m03-sync-first.db');
@@ -72,17 +75,17 @@ test('M03 frozen HTTP contract synchronizes two independent persistent databases
   expect(await second.read()).toEqual([]);
   await new ForegroundSync(second, url, request).synchronize();
   expect(await second.read()).toEqual(final);
-  expect(fixture.state()).toEqual({items: final, nextTimestamp: 1700000104000, requests: [
+  expect(fixture.state()).toEqual({items: final, tombstones: [deleted], nextTimestamp: 1700000104000, requests: [
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: seeds}},
     {method: 'POST', path: '/items', body: {id: 'device-001', title: 'Gamma', completed: false}, status: 201, response: {item: gamma}},
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, ...seeds]}},
-    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Alpha synced'}, status: 200, response: {item: renamed}},
+    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Alpha synced', baseVersion: 1}, status: 200, response: {item: renamed}},
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, renamed, seeds[1]]}},
-    {method: 'PATCH', path: '/items/remote-001', body: {completed: true}, status: 200, response: {item: completed}},
+    {method: 'PATCH', path: '/items/remote-001', body: {completed: true, baseVersion: 2}, status: 200, response: {item: completed}},
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, completed, seeds[1]]}},
-    {method: 'DELETE', path: '/items/remote-002', body: null, status: 200, response: {deletedId: 'remote-002'}},
-    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final}},
-    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final}},
+    {method: 'DELETE', path: '/items/remote-002', body: {baseVersion: 1}, status: 200, response: {deletedId: 'remote-002'}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final, tombstones: [deleted]}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final, tombstones: [deleted]}},
   ]});
   console.info('M03_HTTP_CONVERGENCE', JSON.stringify(fixture.state()));
   closeDatabases();
@@ -232,15 +235,16 @@ test('M05 fixed four offline mutations survive reopening and drain as four accep
   const edited = {...seeds[0], title: 'Queued edit', version: 2, updatedAt: 1700000301000};
   const done = {...edited, completed: true, version: 3, updatedAt: 1700000302000};
   const expected = [created, done];
+  const tombstone = {...deleted, updatedAt: 1700000303000};
   console.info('M05_OFFLINE_RESTART', JSON.stringify({beforeRestart: local, afterDrain: await reopened.read(), remote: fixture.state()}));
   expect(await reopened.read()).toEqual(expected);
-  expect(fixture.state()).toEqual({items: expected, nextTimestamp: 1700000304000, requests: [
+  expect(fixture.state()).toEqual({items: expected, tombstones: [tombstone], nextTimestamp: 1700000304000, requests: [
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: seeds}},
     {method: 'POST', path: '/items', body: {id: 'device-001', title: 'Gamma', completed: false}, status: 201, response: {item: created}},
-    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Queued edit'}, status: 200, response: {item: edited}},
-    {method: 'PATCH', path: '/items/remote-001', body: {completed: true}, status: 200, response: {item: done}},
-    {method: 'DELETE', path: '/items/remote-002', body: null, status: 200, response: {deletedId: 'remote-002'}},
-    {method: 'GET', path: '/items', body: null, status: 200, response: {items: expected}},
+    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Queued edit', baseVersion: 1}, status: 200, response: {item: edited}},
+    {method: 'PATCH', path: '/items/remote-001', body: {completed: true, baseVersion: 2}, status: 200, response: {item: done}},
+    {method: 'DELETE', path: '/items/remote-002', body: {baseVersion: 1}, status: 200, response: {deletedId: 'remote-002'}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: expected, tombstones: [tombstone]}},
   ]});
 });
 
@@ -261,18 +265,18 @@ test('M05 one failed offline foreground drain retains every ordered intent for r
   const pending = await store.readPending();
   expect(pending).toEqual([
     {sequence: 1, kind: 'create', itemId: 'device-001', payload: {id: 'device-001', title: 'Gamma', completed: false}},
-    {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
-    {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
-    {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
+    {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit', baseVersion: 1}},
+    {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true, baseVersion: 1}},
+    {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: {baseVersion: 1}},
   ].map(operation => ({...operation, clientMutationId: expect.any(String),
-    payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null})));
+    payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null, dispatched: false})));
   await expect(sync.synchronize()).rejects.toThrow('Network request failed');
   expect(fixture.state().requests).toHaveLength(1);
   expect(fixture.state().items).toEqual(seeds);
   closeDatabases();
   const reopened = await openItemStore();
   expect(await reopened.read()).toEqual(local);
-  expect(await reopened.readPending()).toEqual(pending);
+  expect(await reopened.readPending()).toEqual(pending.map((operation, index) => ({...operation, dispatched: index === 0})));
 });
 
 const m06 = require('../verification/M06-inputs.json');
@@ -283,14 +287,15 @@ test('M06 lost acknowledgment resends the exact durable identity and returns the
   const local = await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
   const pending = await store.readPending();
   expect(pending).toEqual([{sequence: 1, kind: 'create', itemId: 'crash-001', payload: m06.payload,
-    clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null}]);
+    clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null, dispatched: false}]);
   await expect(new ForegroundSync(store, url, request).synchronize()).rejects.toThrow('Response closed before full body');
-  expect(await store.readPending()).toEqual(pending);
+  const sent = pending.map(operation => ({...operation, dispatched: true}));
+  expect(await store.readPending()).toEqual(sent);
   expect(await store.read()).toEqual(local);
   expect(fixture.mutationState().appliedCount).toBe(1);
   closeDatabases();
   const reopened = await openItemStore();
-  expect(await reopened.readPending()).toEqual(pending);
+  expect(await reopened.readPending()).toEqual(sent);
   expect(await reopened.read()).toEqual(local);
   await new ForegroundSync(reopened, url, request).synchronize();
   expect(await reopened.read()).toEqual([m06.canonicalItem]);
@@ -319,7 +324,7 @@ test('M06 a valid identity collision becomes durable terminal intent; a wrong ha
   const pending = await store.readPending();
   expect(pending[0].payloadHash).toBe(m06.collisionHash);
   await expect(new ForegroundSync(store, url, request).synchronize()).rejects.toThrow('Mutation identity conflict');
-  const terminal = [{...pending[0], terminalError: 'identity_conflict'}];
+  const terminal = [{...pending[0], terminalError: 'identity_conflict', dispatched: true}];
   expect(await store.readPending()).toEqual(terminal);
   const collision = fixture.mutationState();
   expect(collision.conflictCount).toBe(1);
@@ -349,6 +354,231 @@ test('M06 an invalid remote acknowledgment never silently removes durable intent
   const invalid: JsonRequest = async () => ({status: 201, json: async () => ({item: {...m06.canonicalItem, id: 'wrong'}})});
   await expect(new ForegroundSync(store, url, invalid).synchronize()).rejects.toThrow('belongs to another Item');
   closeDatabases();
-  expect(await (await openItemStore()).readPending()).toEqual(pending);
+  expect(await (await openItemStore()).readPending()).toEqual(pending.map(operation => ({...operation, dispatched: true})));
+  expect(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement).toBeNull();
+});
+
+const m07 = require('../verification/M07-inputs.json');
+type ConflictCase = {case: string; kind: 'rename' | 'delete'; clientMutationId: string;
+  envelope: {method: string; path: string; payload: unknown; wireBody: unknown; payloadHash: string};
+  remoteControl: {path: string; body: unknown}; conflictResponse: unknown; conflictEvidence: MutationConflict;
+  canonicalItems: Item[]; canonicalTombstones: unknown[]};
+
+test.each<ConflictCase>(m07.cases)('M07 case $case preserves original conflict across reopen, without ordinary retries', async scenario => {
+  await control('/__m07-reset', {});
+  const store = await openItemStore(undefined, () => scenario.clientMutationId);
+  const sync = new ForegroundSync(store, url, request);
+  await sync.synchronize();
+  expect(await store.read()).toEqual([m07.seed]);
+  await store.mutate(scenario.kind === 'delete' ? {type: 'delete', id: m07.seed.id}
+    : {type: 'rename', id: m07.seed.id, title: 'Local attempt', now: m07.seed.updatedAt});
+  expect(await store.readPending()).toEqual([{...scenario.conflictEvidence.intent, dispatched: false}]);
+  // This remote mutation finishes before any client dispatch, not a timed race.
+  await control(scenario.remoteControl.path, scenario.remoteControl.body);
+  await sync.synchronize();
+  expect(await store.read()).toEqual(scenario.canonicalItems);
+  expect(await store.readPending()).toEqual([]);
+  expect(await store.readConflicts()).toEqual([scenario.conflictEvidence]);
   expect(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement).toBeNull();
+  const canonicalResponse = {items: scenario.canonicalItems,
+    ...(scenario.canonicalTombstones.length ? {tombstones: scenario.canonicalTombstones} : {})};
+  const trace: Trace[] = [
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [m07.seed]}},
+    {method: scenario.envelope.method, path: scenario.envelope.path, body: scenario.envelope.payload,
+      status: 409, response: scenario.conflictResponse},
+    {method: 'GET', path: '/items', body: null, status: 200, response: canonicalResponse},
+  ];
+  expect(fixture.state().requests).toEqual(trace);
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 0, duplicateCount: 0,
+    attempts: [{wireBody: scenario.envelope.wireBody, declaredHash: scenario.envelope.payloadHash,
+      actualHash: scenario.envelope.payloadHash, status: 409, response: scenario.conflictResponse}]});
+  const originalEvidence = JSON.parse(JSON.stringify(fixture.mutationState()));
+  closeDatabases();
+  const reopened = await openItemStore(undefined, () => m07.explicitEnvelope.clientMutationId);
+  expect(await reopened.readConflicts()).toEqual([scenario.conflictEvidence]);
+  expect(await reopened.read()).toEqual(scenario.canonicalItems);
+  await new ForegroundSync(reopened, url, request).synchronize();
+  trace.push({method: 'GET', path: '/items', body: null, status: 200, response: canonicalResponse});
+  expect(fixture.mutationState()).toEqual(originalEvidence);
+  if (scenario.case === 'A') {
+    await reopened.mutate({type: 'rename', id: m07.seed.id, title: 'Explicit edit', now: m07.explicitItem.updatedAt});
+    const explicit = m07.explicitEnvelope;
+    expect(await reopened.readPending()).toEqual([{sequence: 2, kind: 'rename', itemId: m07.seed.id,
+      payload: explicit.payload, clientMutationId: explicit.clientMutationId, payloadHash: explicit.payloadHash,
+      terminalError: null, dispatched: false}]);
+    await new ForegroundSync(reopened, url, request).synchronize();
+    expect(await reopened.read()).toEqual([m07.explicitItem]);
+    expect(await reopened.readPending()).toEqual([]);
+    trace.push({method: explicit.method, path: explicit.path, body: explicit.payload, status: 200, response: {item: m07.explicitItem}},
+      {method: 'GET', path: '/items', body: null, status: 200, response: {items: [m07.explicitItem]}});
+    expect(fixture.mutationState().appliedCount).toBe(1);
+  }
+  expect(await reopened.readConflicts()).toEqual([scenario.conflictEvidence]);
+  expect(fixture.state().requests).toEqual(trace);
+  console.info('M07_CONFLICT_CASE', JSON.stringify({case: scenario.case, conflicts: await reopened.readConflicts(),
+    canonical: await reopened.read(), remote: fixture.state(), mutations: fixture.mutationState()}));
+});
+
+async function acknowledgeOwnFirst(store: Awaited<ReturnType<typeof openItemStore>>) {
+  const first = (await store.prepareNext())!;
+  const target = mutationTarget(first.kind, first.itemId);
+  const response = await request(url + target.path, {method: target.method, headers: {'Content-Type': 'application/json'},
+    body: JSON.stringify({...first.payload, clientMutationId: first.clientMutationId, payloadHash: first.payloadHash})});
+  expect(response.status).toBe(200);
+  const result = await response.json() as {item: Item};
+  expect(result).toEqual({item: m07.successorRegression.firstResult});
+  await store.acknowledge(first, result);
+  return first;
+}
+
+async function queuedOwnSuccessor() {
+  await control('/__m07-reset', {});
+  const fixtureInput = m07.successorRegression;
+  const identities = [fixtureInput.firstIdentity, fixtureInput.successorIdentity];
+  const store = await openItemStore(undefined, () => identities.shift()!);
+  await new ForegroundSync(store, url, request).synchronize();
+  await store.mutate({type: 'rename', id: m07.seed.id, title: 'Own predecessor', now: m07.seed.updatedAt});
+  await store.mutate({type: 'toggle', id: m07.seed.id, now: m07.seed.updatedAt});
+  expect((await store.readPending()).map(operation => ({payload: operation.payload, hash: operation.payloadHash}))).toEqual([
+    {payload: fixtureInput.first.payload, hash: fixtureInput.first.payloadHash},
+    {payload: fixtureInput.initialSuccessor.payload, hash: fixtureInput.initialSuccessor.payloadHash},
+  ]);
+  return store;
+}
+
+test('M07 own ACK durably prepares a successor; its lost-ACK replay stays identical despite newer remote state', async () => {
+  const input = m07.successorRegression;
+  const store = await queuedOwnSuccessor();
+  const optimistic = await store.read();
+  await acknowledgeOwnFirst(store);
+  expect(await store.read()).toEqual(optimistic); // No earlier ACK overwrites the later local toggle.
+  closeDatabases();
+  let reopened = await openItemStore();
+  const prepared: PendingMutation = {sequence: 2, kind: 'toggle', itemId: m07.seed.id,
+    payload: input.preparedSuccessor.payload, clientMutationId: input.successorIdentity,
+    payloadHash: input.preparedSuccessor.payloadHash, terminalError: null, dispatched: false};
+  expect(await reopened.readPending()).toEqual([prepared]);
+  await control('/__drop-ack', {clientMutationId: input.successorIdentity});
+  await expect(new ForegroundSync(reopened, url, request).synchronize()).rejects.toThrow('Response closed before full body');
+  const sent = {...prepared, dispatched: true};
+  closeDatabases();
+  reopened = await openItemStore();
+  expect(await reopened.readPending()).toEqual([sent]);
+  expect(await reopened.read()).toEqual(optimistic);
+  await control('/__remote-change', {id: m07.seed.id, title: input.externalAfterLoss.title, updatedAt: input.externalAfterLoss.updatedAt});
+  await new ForegroundSync(reopened, url, request).synchronize();
+  expect(await reopened.readPending()).toEqual([]);
+  expect(await reopened.readConflicts()).toEqual([]);
+  expect(await reopened.read()).toEqual([input.externalAfterLoss]);
+  const ack = JSON.parse(String(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement));
+  expect(ack).toEqual({clientMutationId: input.successorIdentity, payloadHash: sent.payloadHash, status: 200, result: {item: input.successorResult}});
+  const attempts = fixture.mutationState().attempts;
+  expect(attempts).toHaveLength(3);
+  expect(attempts[1]).toMatchObject({wireBody: input.preparedSuccessor.wireBody, actualHash: sent.payloadHash,
+    outcome: 'applied', responseDropped: true, response: {item: input.successorResult}});
+  expect(attempts[2]).toMatchObject({wireBody: input.preparedSuccessor.wireBody, actualHash: sent.payloadHash,
+    outcome: 'duplicate', responseDropped: false, response: {item: input.successorResult}});
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 2, duplicateCount: 1, conflictCount: 0});
+  console.info('M07_OWN_SUCCESSOR_LOST_ACK', JSON.stringify({prepared, sent, ack, final: await reopened.read(), mutations: fixture.mutationState()}));
+});
+
+test('M07 external version never rebases an own-prepared successor; a base-only identity change is terminal', async () => {
+  const input = m07.successorRegression;
+  const store = await queuedOwnSuccessor();
+  await acknowledgeOwnFirst(store);
+  const prepared = (await store.readPending())[0];
+  expect(prepared.payload).toEqual(input.preparedSuccessor.payload);
+  await control('/__remote-change', {id: m07.seed.id, title: input.externalBeforeSuccessor.title, updatedAt: input.externalBeforeSuccessor.updatedAt});
+  closeDatabases();
+  const reopened = await openItemStore();
+  await new ForegroundSync(reopened, url, request).synchronize();
+  expect(await reopened.read()).toEqual([input.externalBeforeSuccessor]);
+  expect(await reopened.readConflicts()).toEqual([{intent: {...prepared, dispatched: true}, reason: 'version_conflict',
+    item: input.externalBeforeSuccessor, tombstone: null}]);
+  expect(fixture.mutationState().attempts[1]).toMatchObject({wireBody: input.preparedSuccessor.wireBody, status: 409,
+    response: {error: 'version_conflict', item: input.externalBeforeSuccessor, tombstone: null}});
+  expect(input.baseOnlyCollision.payloadHash).not.toBe(input.first.payloadHash);
+  expect(mutationHash('PATCH', input.first.path, input.baseOnlyCollision.payload)).toBe(input.baseOnlyCollision.payloadHash);
+  const collision = await request(url + input.baseOnlyCollision.path, {method: 'PATCH', headers: {'Content-Type': 'application/json'},
+    body: JSON.stringify(input.baseOnlyCollision.wireBody)});
+  expect(collision.status).toBe(409);
+  expect(await collision.json()).toEqual({error: 'identity_conflict'});
+  const counts = JSON.parse(JSON.stringify(fixture.mutationState()));
+  await new ForegroundSync(reopened, url, request).synchronize();
+  expect(fixture.mutationState()).toEqual(counts);
+  expect(counts).toMatchObject({appliedCount: 1, duplicateCount: 0, conflictCount: 1});
+  expect(fixture.state().items).toEqual([input.externalBeforeSuccessor]);
+  console.info('M07_EXTERNAL_VERSION_NOT_REBASED', JSON.stringify({prepared, conflicts: await reopened.readConflicts(), mutations: counts}));
+});
+
+test.each([/^UPDATE pending_mutations SET dispatched/, /^END/])('M07 failed first-dispatch persistence at %s sends nothing', async failAt => {
+  const store = await openItemStore(undefined, () => m07.cases[0].clientMutationId);
+  await store.replaceSnapshot([m07.seed]);
+  await store.mutate({type: 'rename', id: m07.seed.id, title: 'Local attempt', now: m07.seed.updatedAt});
+  const pending = await store.readPending();
+  const transport: JsonRequest = jest.fn(() => Promise.reject(new Error('No request before durable dispatch')));
+  failNextSql(failAt);
+  await expect(new ForegroundSync(store, url, transport).synchronize()).rejects.toThrow('injected persistence failure');
+  expect(transport).not.toHaveBeenCalled();
+  closeDatabases();
+  expect(await (await openItemStore()).readPending()).toEqual(pending);
+});
+
+test('M07 schema4 keeps original unversioned intent blocked and never infers canonical versions from optimistic caches', async () => {
+  // Also cover M06 ACK success followed by GET failure: an empty queue does NOT
+  // establish canonical provenance for the remaining optimistic Item fields.
+  for (const hasPending of [true, false]) {
+    await control('/__m07-reset', {});
+    const name = `m07-legacy-${hasPending}.db`;
+    const db = connection(name);
+    db.exec(`
+      CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL, completed INTEGER NOT NULL, version INTEGER NOT NULL, updated_at INTEGER NOT NULL);
+      CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY, next_id INTEGER NOT NULL);
+      INSERT INTO local_identity VALUES (1, 2);
+      CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY, last_successful_refresh_at INTEGER, last_acknowledgement TEXT);
+      INSERT INTO sync_metadata VALUES (1, 1700000000000, NULL);
+      CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL, item_id TEXT NOT NULL,
+        payload TEXT CHECK((kind = 'delete' AND payload IS NULL) OR (kind != 'delete' AND payload IS NOT NULL)),
+        client_mutation_id TEXT NOT NULL, payload_hash TEXT NOT NULL, terminal_error TEXT);
+      CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id);
+      PRAGMA user_version = 4;
+    `);
+    const original = m07.legacyUpgrade.original;
+    const optimistic = {...m07.seed, title: 'Legacy attempt', version: 2};
+    db.prepare('INSERT INTO items VALUES (?, ?, ?, ?, ?)').run(optimistic.id, optimistic.title, 0, 2, optimistic.updatedAt);
+    if (hasPending) {
+      db.prepare('INSERT INTO pending_mutations VALUES (?, ?, ?, ?, ?, ?, ?)')
+        .run(7, 'rename', m07.seed.id, JSON.stringify(original.payload), original.clientMutationId, original.payloadHash, null);
+    } else {
+      db.prepare('UPDATE sync_metadata SET last_acknowledgement = ?').run(JSON.stringify({clientMutationId: original.clientMutationId,
+        payloadHash: original.payloadHash, status: 200, result: {item: {...optimistic, updatedAt: m07.nextAcceptedTimestamp}}}));
+    }
+    const metadata = db.prepare('SELECT * FROM sync_metadata').get();
+    const makeIdentity = jest.fn(() => m07.explicitEnvelope.clientMutationId);
+    const store = await openItemStore(name, makeIdentity);
+    expect(makeIdentity).not.toHaveBeenCalled();
+    expect(await store.read()).toEqual([optimistic]);
+    expect(await store.readPending()).toEqual([]);
+    expect(db.prepare('SELECT * FROM remote_versions').all()).toEqual([]);
+    expect(db.prepare('SELECT * FROM sync_metadata').get()).toEqual(metadata);
+    const conflicts = hasPending ? [{intent: {sequence: 7, kind: 'rename', itemId: m07.seed.id, payload: original.payload,
+      clientMutationId: original.clientMutationId, payloadHash: original.payloadHash, terminalError: null, dispatched: true},
+      reason: 'unversioned_legacy', item: null, tombstone: null}] : [];
+    expect(await store.readConflicts()).toEqual(conflicts);
+    closeDatabases();
+    const reopened = await openItemStore(name, makeIdentity);
+    expect(await reopened.readConflicts()).toEqual(conflicts);
+    await new ForegroundSync(reopened, url, request).synchronize();
+    expect(await reopened.read()).toEqual([m07.seed]);
+    expect(fixture.mutationState().attempts).toEqual([]);
+    expect(fixture.state().requests).toEqual([{method: 'GET', path: '/items', body: null, status: 200, response: {items: [m07.seed]}}]);
+    await reopened.mutate({type: 'rename', id: m07.seed.id, title: 'Explicit edit', now: m07.explicitItem.updatedAt});
+    const fresh = (await reopened.readPending())[0];
+    expect(fresh.clientMutationId).toBe(m07.explicitEnvelope.clientMutationId);
+    expect(fresh.payload).toEqual({title: 'Explicit edit', baseVersion: 1});
+    expect(fresh.payloadHash).toBe(mutationHash('PATCH', '/items/conflict-001', fresh.payload));
+    expect(await reopened.readConflicts()).toEqual(conflicts);
+    console.info('M07_LEGACY_UNKNOWN_PROVENANCE', JSON.stringify({hasPending, metadata, conflicts, fresh}));
+    closeDatabases();
+  }
 });
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index 3dfe55a..87655e4 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -22,6 +22,9 @@ class MainActivity : ReactActivity() {
                         putString("testIdentityPrefix", "crash")
                         putString("testMutationIdentity", "m06-create-001")
                     }
+                    intent.getStringExtra("m07MutationIdentity")?.let {
+                        putString("testMutationIdentity", it)
+                    }
                 }
             }
         }
diff --git a/scripts/verify_m02.py b/scripts/verify_m02.py
index b5c8e24..f221078 100644
--- a/scripts/verify_m02.py
+++ b/scripts/verify_m02.py
@@ -99,11 +99,11 @@ def main():
         assert path.exists(), "Native database file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as database:
             assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert database.execute("PRAGMA user_version").fetchone()[0] == 4
+            assert database.execute("PRAGMA user_version").fetchone()[0] == 5
             assert [column[1] for column in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in database.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid")]
-            state = {"schema_version": 4, "items": items,
+            state = {"schema_version": 5, "items": items,
                      "next_id": database.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(state, indent=2))
         return state
diff --git a/scripts/verify_m03.py b/scripts/verify_m03.py
index b73db64..ba5e0c4 100644
--- a/scripts/verify_m03.py
+++ b/scripts/verify_m03.py
@@ -26,23 +26,24 @@ GAMMA = {"id": "device-001", "title": "Gamma", "completed": False, "version": 1,
 RENAMED = {**SEEDS[0], "title": "Alpha synced", "version": 2, "updatedAt": 1700000101000}
 COMPLETED = {**RENAMED, "completed": True, "version": 3, "updatedAt": 1700000102000}
 FINAL = [GAMMA, COMPLETED]
+TOMBSTONE = {"id": "remote-002", "version": 2, "updatedAt": 1700000103000, "deleted": True}
 
 
 def expected_trace():
     def event(method, path, body, status, response):
         return {"method": method, "path": path, "body": body, "status": status, "response": response}
-    def get(items):
-        return event("GET", "/items", None, 200, {"items": items})
+    def get(items, deleted=False):
+        return event("GET", "/items", None, 200, {"items": items, **({"tombstones": [TOMBSTONE]} if deleted else {})})
     return [
         get(SEEDS),
         event("POST", "/items", {"id": "device-001", "title": "Gamma", "completed": False}, 201, {"item": GAMMA}),
         get([GAMMA, *SEEDS]),
-        event("PATCH", "/items/remote-001", {"title": "Alpha synced"}, 200, {"item": RENAMED}),
+        event("PATCH", "/items/remote-001", {"title": "Alpha synced", "baseVersion": 1}, 200, {"item": RENAMED}),
         get([GAMMA, RENAMED, SEEDS[1]]),
-        event("PATCH", "/items/remote-001", {"completed": True}, 200, {"item": COMPLETED}),
+        event("PATCH", "/items/remote-001", {"completed": True, "baseVersion": 2}, 200, {"item": COMPLETED}),
         get([GAMMA, COMPLETED, SEEDS[1]]),
-        event("DELETE", "/items/remote-002", None, 200, {"deletedId": "remote-002"}),
-        get(FINAL), get(FINAL),
+        event("DELETE", "/items/remote-002", {"baseVersion": 1}, 200, {"deletedId": "remote-002"}),
+        get(FINAL, True), get(FINAL, True),
     ]
 
 
@@ -138,11 +139,11 @@ def main():
         assert path.exists(), "Native SQLite file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert db.execute("PRAGMA user_version").fetchone()[0] == 4
+            assert db.execute("PRAGMA user_version").fetchone()[0] == 5
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
-            value = {"items": items, "schema_version": 4,
+            value = {"items": items, "schema_version": 5,
                      "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
         return value
@@ -264,7 +265,7 @@ def main():
         result["first_preserved"] = database("first-preserved-final", "m03-first.db")
         assert result["first_preserved"] == result["before_kill"]
         result["remote"] = remote()
-        assert result["remote"] == {"items": FINAL, "nextTimestamp": 1700000104000, "requests": expected_trace()}
+        assert result["remote"] == {"items": FINAL, "tombstones": [TOMBSTONE], "nextTimestamp": 1700000104000, "requests": expected_trace()}
         (evidence / "remote-final.json").write_text(json.dumps(result["remote"], indent=2) + "\n")
         assert os.getpid() == result["host_pid"]
         result["status"] = "PASS"
diff --git a/scripts/verify_m04.py b/scripts/verify_m04.py
index b0810b2..5c75c06 100644
--- a/scripts/verify_m04.py
+++ b/scripts/verify_m04.py
@@ -116,7 +116,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (1 if args.baseline else 4), schema
+            assert schema == (1 if args.baseline else 5), schema
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
diff --git a/scripts/verify_m05.py b/scripts/verify_m05.py
index 50dab68..836d280 100644
--- a/scripts/verify_m05.py
+++ b/scripts/verify_m05.py
@@ -27,11 +27,12 @@ CREATED = {"id": "device-001", "title": "Gamma", "completed": False, "version":
 RENAMED = {**SEEDS[0], "title": "Queued edit", "version": 2, "updatedAt": 1700000301000}
 COMPLETED = {**RENAMED, "completed": True, "version": 3, "updatedAt": 1700000302000}
 FINAL = [CREATED, COMPLETED]
+TOMBSTONE = {"id": "remote-002", "version": 2, "updatedAt": 1700000303000, "deleted": True}
 INTENTS = [
     {"sequence": 1, "kind": "create", "itemId": "device-001", "payload": {"id": "device-001", "title": "Gamma", "completed": False}},
-    {"sequence": 2, "kind": "rename", "itemId": "remote-001", "payload": {"title": "Queued edit"}},
-    {"sequence": 3, "kind": "toggle", "itemId": "remote-001", "payload": {"completed": True}},
-    {"sequence": 4, "kind": "delete", "itemId": "remote-002", "payload": None},
+    {"sequence": 2, "kind": "rename", "itemId": "remote-001", "payload": {"title": "Queued edit", "baseVersion": 1}},
+    {"sequence": 3, "kind": "toggle", "itemId": "remote-001", "payload": {"completed": True, "baseVersion": 1}},
+    {"sequence": 4, "kind": "delete", "itemId": "remote-002", "payload": {"baseVersion": 1}},
 ]
 
 
@@ -45,9 +46,9 @@ def expected_trace(baseline):
     return [get(SEEDS),
             event("POST", "/items", INTENTS[0]["payload"], 201, {"item": CREATED}),
             event("PATCH", "/items/remote-001", INTENTS[1]["payload"], 200, {"item": RENAMED}),
-            event("PATCH", "/items/remote-001", INTENTS[2]["payload"], 200, {"item": COMPLETED}),
-            event("DELETE", "/items/remote-002", None, 200, {"deletedId": "remote-002"}),
-            get(FINAL)]
+            event("PATCH", "/items/remote-001", {**INTENTS[2]["payload"], "baseVersion": 2}, 200, {"item": COMPLETED}),
+            event("DELETE", "/items/remote-002", INTENTS[3]["payload"], 200, {"deletedId": "remote-002"}),
+            event("GET", "/items", None, 200, {"items": FINAL, "tombstones": [TOMBSTONE]})]
 
 
 def main():
@@ -147,7 +148,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (2 if args.baseline else 4), schema
+            assert schema == (2 if args.baseline else 5), schema
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
             tables = [row[0] for row in db.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
@@ -294,6 +295,7 @@ def main():
             assert result["after_drain"]["pending"] == []
         result["remote"] = remote()
         assert result["remote"] == {"items": expected, "nextTimestamp": 1700000300000 if args.baseline else 1700000304000,
+                                    **({} if args.baseline else {"tombstones": [TOMBSTONE]}),
                                     "requests": expected_trace(args.baseline)}
         (evidence / "remote-final.json").write_text(json.dumps(result["remote"], indent=2) + "\n")
         result["accepted_operations"] = [event for event in result["remote"]["requests"] if event["method"] != "GET"]
diff --git a/scripts/verify_m06.py b/scripts/verify_m06.py
index 30fbea9..bdbcbe3 100644
--- a/scripts/verify_m06.py
+++ b/scripts/verify_m06.py
@@ -124,7 +124,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (3 if args.baseline else 4), schema
+            assert schema == (3 if args.baseline else 5), schema
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
             columns = "sequence, kind, item_id, payload" + ("" if args.baseline else ", client_mutation_id, payload_hash, terminal_error")
diff --git a/src/App.tsx b/src/App.tsx
index 9d14ffc..1ceb0d3 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -29,6 +29,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const [lastSuccessfulRefreshAt, setLastSuccessfulRefreshAt] = useState<number | null>(null);
   const [pendingCount, setPendingCount] = useState<number | null>(null);
   const [identityBlocked, setIdentityBlocked] = useState(false);
+  const [conflictCount, setConflictCount] = useState<number | null>(null);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
@@ -48,6 +49,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       const saved = await opened.read();
       const lastRefresh = await opened.readLastSuccessfulRefresh();
       const pending = await opened.readPending();
+      const conflicts = await opened.readConflicts();
       if (active) {
         store.current = opened;
         sync.current = createSync(opened, testIdentityPrefix, testRefreshClock);
@@ -55,6 +57,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         setLastSuccessfulRefreshAt(lastRefresh);
         setPendingCount(pending.length);
         setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
+        setConflictCount(conflicts.length);
         setRefresh({status: 'stale'});
         setReady(true);
       }
@@ -71,8 +74,9 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       const pending = await store.current!.readPending();
       setPendingCount(pending.length);
       setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
+      setConflictCount((await store.current!.readConflicts()).length);
     }
-    catch {setPendingCount(null);} // A failed status read must not claim a committed edit was unsaved.
+    catch {setPendingCount(null); setConflictCount(null);} // A failed status read must not claim a committed edit was unsaved.
   }
 
   async function mutate(action: ItemMutation): Promise<boolean> {
@@ -109,6 +113,9 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       setLastSuccessfulRefreshAt(lastRefresh);
       setRefresh({status: 'fresh'});
     } catch (reason) {
+      // A conflict can commit its canonical winner before a later GET fails.
+      // Show that committed state, while retaining the refresh error/time.
+      try {setItems(await store.current.read());} catch { /* Keep the last confirmed list. */ }
       setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});
     } finally {
       await reloadPending();
@@ -153,6 +160,9 @@ export default function App({openStore = openItemStore, createSync = defaultSync
           {pendingCount === null ? 'Pending upload count unavailable' : `Pending uploads: ${pendingCount}`}
         </Text>
         {identityBlocked && <Text accessibilityRole="alert">Upload blocked: mutation identity conflict. It will not be retried.</Text>}
+        {conflictCount !== null && conflictCount > 0 && <Text accessibilityRole="alert" accessibilityLabel={`Conflicts preserved: ${conflictCount}`}>
+          Conflicts preserved: {conflictCount}. Canonical state wins after refresh. Original attempts are saved and will not retry. Edit a current Item to make a new attempt.
+        </Text>}
         {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
       </>}
       <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
diff --git a/src/itemStore.ts b/src/itemStore.ts
index 388ae1c..d6a3944 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -3,7 +3,7 @@ import {Item, ItemAction, itemsReducer} from './items';
 import {mutationHash, mutationTarget, newMutationIdentity} from './mutationProtocol';
 
 export const DATABASE_NAME = 'items.db';
-export const SCHEMA_VERSION = 4;
+export const SCHEMA_VERSION = 5;
 
 export type ItemRow = {
   id: string;
@@ -34,27 +34,42 @@ export type ItemMutation = Exclude<ItemAction, {type: 'create'}>
 
 type UploadOperation =
   | {kind: 'create'; itemId: string; payload: {id: string; title: string; completed: boolean}}
-  | {kind: 'rename'; itemId: string; payload: {title: string}}
-  | {kind: 'toggle'; itemId: string; payload: {completed: boolean}}
-  | {kind: 'delete'; itemId: string; payload: null};
+  // Missing bases exist only in preserved, blocked schema4 evidence.
+  | {kind: 'rename'; itemId: string; payload: {title: string; baseVersion?: number}}
+  | {kind: 'toggle'; itemId: string; payload: {completed: boolean; baseVersion?: number}}
+  | {kind: 'delete'; itemId: string; payload: {baseVersion: number} | null};
 
 export type PendingMutation = UploadOperation & {
   sequence: number;
   clientMutationId: string;
   payloadHash: string;
   terminalError: 'identity_conflict' | null;
+  dispatched: boolean;
 };
 
 export type MutationResult = {item: Item} | {deletedId: string};
+export type Tombstone = {id: string; version: number; updatedAt: number; deleted: true};
+export type MutationConflict = {
+  intent: PendingMutation;
+  reason: 'version_conflict' | 'unversioned_legacy';
+  item: Item | null;
+  tombstone: Tombstone | null;
+};
+// A DELETE receipt gives its accepted base+1, but no tombstone timestamp. Keep
+// that time unknown until observed, rather than inventing a canonical tombstone.
+type RemoteVersion = {id: string; version: number; updatedAt: number | null; deleted: boolean; item: Item | null};
 
 export interface ItemStore {
   read(): Promise<Item[]>;
   readLastSuccessfulRefresh(): Promise<number | null>;
   readPending(): Promise<PendingMutation[]>;
+  readConflicts(): Promise<MutationConflict[]>;
+  prepareNext(): Promise<PendingMutation | null>;
   mutate(action: ItemMutation, identityPrefix?: string): Promise<Item[]>;
   acknowledge(operation: PendingMutation, result: MutationResult): Promise<void>;
   rejectIdentity(operation: PendingMutation): Promise<void>;
-  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void>;
+  rejectVersion(operation: PendingMutation, item: Item | null, tombstone: Tombstone | null): Promise<void>;
+  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number, tombstones?: Tombstone[]): Promise<void>;
 }
 
 function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
@@ -67,13 +82,16 @@ function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
   });
 }
 
-function decodeOperation(row: {sequence: number; kind: string; item_id: string; payload: string | null}) {
+function decodeOperation(row: {sequence: number; kind: string; item_id: string; payload: string | null}, allowUnversioned = false) {
   const payload = row.payload === null ? null : JSON.parse(row.payload);
   const title = payload && typeof payload.title === 'string' && payload.title.trim();
-  const validPayload = row.kind === 'delete' ? payload === null
+  const hasBase = payload && Object.prototype.hasOwnProperty.call(payload, 'baseVersion');
+  const validBase = hasBase && Number.isSafeInteger(payload.baseVersion) && payload.baseVersion >= 0;
+  const fields = hasBase ? 2 : 1;
+  const validPayload = row.kind === 'delete' ? (validBase && Object.keys(payload).length === 1) || (allowUnversioned && payload === null)
     : row.kind === 'create' ? title && payload.id === row.item_id && typeof payload.completed === 'boolean' && Object.keys(payload).length === 3
-      : row.kind === 'rename' ? title && Object.keys(payload).length === 1
-        : row.kind === 'toggle' && payload && typeof payload.completed === 'boolean' && Object.keys(payload).length === 1;
+      : (validBase || (allowUnversioned && !hasBase)) && (row.kind === 'rename' ? title && Object.keys(payload).length === fields
+        : row.kind === 'toggle' && payload && typeof payload.completed === 'boolean' && Object.keys(payload).length === fields);
   if (!Number.isSafeInteger(row.sequence) || row.sequence < 1
       || typeof row.item_id !== 'string' || !row.item_id || !validPayload) {
     throw new Error('Invalid pending mutation in the local database');
@@ -89,24 +107,66 @@ function identityFor(operation: UploadOperation, makeIdentity: () => string) {
 }
 
 function readPending(tx: SQLTransaction, callback: (pending: PendingMutation[]) => void) {
-  tx.executeSql('SELECT sequence, kind, item_id, payload, client_mutation_id, payload_hash, terminal_error FROM pending_mutations ORDER BY sequence', [], (_, result) => {
+  tx.executeSql('SELECT sequence, kind, item_id, payload, client_mutation_id, payload_hash, terminal_error, dispatched FROM pending_mutations ORDER BY sequence', [], (_, result) => {
     const pending: PendingMutation[] = [];
     for (let i = 0; i < result.rows.length; i++) {
-      const row = result.rows.item(i);
-      const operation = decodeOperation(row);
-      const target = mutationTarget(operation.kind, operation.itemId);
-      if (typeof row.client_mutation_id !== 'string' || !row.client_mutation_id
-          || row.payload_hash !== mutationHash(target.method, target.path, operation.payload)
-          || (row.terminal_error !== null && row.terminal_error !== 'identity_conflict')) {
-        throw new Error('Invalid durable mutation identity or payload hash');
-      }
-      pending.push({...operation, clientMutationId: row.client_mutation_id,
-        payloadHash: row.payload_hash, terminalError: row.terminal_error});
+      pending.push(decodeMutation(result.rows.item(i)));
     }
     callback(pending);
   });
 }
 
+type MutationRow = {sequence: number; kind: string; item_id: string; payload: string | null;
+  client_mutation_id: string; payload_hash: string; terminal_error: 'identity_conflict' | null; dispatched: number};
+
+function decodeMutation(row: MutationRow, allowUnversioned = false): PendingMutation {
+  const operation = decodeOperation(row, allowUnversioned || row.terminal_error === 'identity_conflict');
+  const target = mutationTarget(operation.kind, operation.itemId);
+  if (typeof row.client_mutation_id !== 'string' || !row.client_mutation_id
+      || row.payload_hash !== mutationHash(target.method, target.path, operation.payload)
+      || (row.terminal_error !== null && row.terminal_error !== 'identity_conflict')
+      || (row.dispatched !== 0 && row.dispatched !== 1)) {
+    throw new Error('Invalid durable mutation identity or payload hash');
+  }
+  return {...operation, clientMutationId: row.client_mutation_id, payloadHash: row.payload_hash,
+    terminalError: row.terminal_error, dispatched: row.dispatched === 1};
+}
+
+function matchesHead(first: PendingMutation | undefined, operation: PendingMutation) {
+  return first && first.sequence === operation.sequence && first.clientMutationId === operation.clientMutationId
+    && first.payloadHash === operation.payloadHash && first.terminalError === null && first.dispatched;
+}
+
+function writeConflict(tx: SQLTransaction, conflict: MutationConflict) {
+  tx.executeSql('INSERT INTO mutation_conflicts (client_mutation_id, intent, reason, item, tombstone) VALUES (?, ?, ?, ?, ?)',
+    [conflict.intent.clientMutationId, JSON.stringify(conflict.intent), conflict.reason,
+      conflict.item === null ? null : JSON.stringify(conflict.item), conflict.tombstone === null ? null : JSON.stringify(conflict.tombstone)]);
+}
+
+function readVersions(tx: SQLTransaction, callback: (versions: Map<string, RemoteVersion>) => void) {
+  tx.executeSql('SELECT id, version, updated_at, deleted, canonical_item FROM remote_versions', [], (_, result) => {
+    const versions = new Map<string, RemoteVersion>();
+    for (let i = 0; i < result.rows.length; i++) {
+      const row = result.rows.item(i);
+      versions.set(row.id, {id: row.id, version: row.version, updatedAt: row.updated_at, deleted: row.deleted === 1,
+        item: row.canonical_item === null ? null : JSON.parse(row.canonical_item)});
+    }
+    callback(versions);
+  });
+}
+
+function acceptsVersion(known: RemoteVersion | undefined, incoming: RemoteVersion) {
+  return !known || incoming.version > known.version
+    || (incoming.version === known.version && (!known.deleted || incoming.deleted));
+}
+
+function writeVersion(tx: SQLTransaction, value: RemoteVersion) {
+  if (value.version > 0) {
+    tx.executeSql('INSERT OR REPLACE INTO remote_versions (id, version, updated_at, deleted, canonical_item) VALUES (?, ?, ?, ?, ?)',
+      [value.id, value.version, value.updatedAt, Number(value.deleted), value.item === null ? null : JSON.stringify(value.item)]);
+  }
+}
+
 // Resolve only in the transaction's success callback, after native COMMIT.
 // A failed statement/commit rolls back and must not publish its candidate state.
 class SqliteItemStore implements ItemStore {
@@ -141,6 +201,40 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
+  readConflicts(): Promise<MutationConflict[]> {
+    return new Promise((resolve, reject) => {
+      const conflicts: MutationConflict[] = [];
+      this.database.readTransaction(tx => {
+        tx.executeSql('SELECT intent, reason, item, tombstone FROM mutation_conflicts ORDER BY rowid', [], (_, result) => {
+          for (let i = 0; i < result.rows.length; i++) {
+            const row = result.rows.item(i);
+            const saved = JSON.parse(row.intent) as PendingMutation;
+            if (row.reason !== 'version_conflict' && row.reason !== 'unversioned_legacy') {throw new Error('Invalid conflict reason');}
+            const intent = decodeMutation({sequence: saved.sequence, kind: saved.kind, item_id: saved.itemId,
+              payload: saved.payload === null ? null : JSON.stringify(saved.payload), client_mutation_id: saved.clientMutationId,
+              payload_hash: saved.payloadHash, terminal_error: saved.terminalError, dispatched: Number(saved.dispatched)}, row.reason === 'unversioned_legacy');
+            conflicts.push({intent, reason: row.reason, item: row.item === null ? null : JSON.parse(row.item),
+              tombstone: row.tombstone === null ? null : JSON.parse(row.tombstone)});
+          }
+        });
+      }, reject, () => resolve(conflicts));
+    });
+  }
+
+  prepareNext(): Promise<PendingMutation | null> {
+    return new Promise((resolve, reject) => {
+      let next: PendingMutation | null = null;
+      this.database.transaction(tx => readPending(tx, pending => {
+        next = pending[0] ?? null;
+        if (next && next.terminalError === null && !next.dispatched) {
+          // Commit BEFORE fetch. A failed response cannot prove it was never sent.
+          tx.executeSql('UPDATE pending_mutations SET dispatched = 1 WHERE sequence = ?', [next.sequence]);
+          next = {...next, dispatched: true};
+        }
+      }), reject, () => resolve(next));
+    });
+  }
+
   acknowledge(operation: PendingMutation, result: MutationResult): Promise<void> {
     const resultId = 'item' in result ? result.item.id : result.deletedId;
     if (resultId !== operation.itemId || (operation.kind === 'delete') !== ('deletedId' in result)) {
@@ -150,8 +244,7 @@ class SqliteItemStore implements ItemStore {
       this.database.transaction(tx => {
         readPending(tx, pending => {
           const first = pending[0];
-          if (!first || first.sequence !== operation.sequence || first.clientMutationId !== operation.clientMutationId
-              || first.payloadHash !== operation.payloadHash || first.terminalError !== null) {
+          if (!matchesHead(first, operation)) {
             throw new Error('Acknowledgment does not match the next pending mutation');
           }
           // Record the actual result before dequeuing in the SAME native commit.
@@ -161,6 +254,31 @@ class SqliteItemStore implements ItemStore {
           tx.executeSql('UPDATE sync_metadata SET last_acknowledgement = ? WHERE singleton = 1', [JSON.stringify(acknowledgement)]);
           tx.executeSql('DELETE FROM pending_mutations WHERE sequence = ? AND client_mutation_id = ? AND payload_hash = ?',
             [first.sequence, first.clientMutationId, first.payloadHash]);
+          if ('item' in result) {
+            const accepted = {...result.item, deleted: false, item: result.item};
+            readVersions(tx, versions => {
+              if (acceptsVersion(versions.get(accepted.id), accepted)) {writeVersion(tx, accepted);}
+            });
+            const successor = pending.slice(1).find(candidate => candidate.itemId === first.itemId);
+            if (successor && successor.kind !== 'create' && !successor.dispatched && successor.terminalError === null) {
+              // Only this acknowledged OWN predecessor can prepare a never-sent
+              // successor. External GET/conflict versions never rebase a queue.
+              const payload = {...successor.payload, baseVersion: result.item.version};
+              const target = mutationTarget(successor.kind, successor.itemId);
+              const hash = mutationHash(target.method, target.path, payload);
+              tx.executeSql('UPDATE pending_mutations SET payload = ?, payload_hash = ? WHERE sequence = ? AND client_mutation_id = ? AND payload_hash = ? AND dispatched = 0',
+                [JSON.stringify(payload), hash, successor.sequence, successor.clientMutationId, successor.payloadHash], (_, changed) => {
+                  if (changed.rowsAffected !== 1) {throw new Error('Own successor preparation did not match durable intent');}
+                });
+            }
+          } else if (first.kind === 'delete' && first.payload !== null) {
+            const deletion: RemoteVersion = {id: first.itemId, version: first.payload.baseVersion + 1,
+              updatedAt: null, deleted: true, item: null};
+            readVersions(tx, versions => {
+              const known = versions.get(first.itemId);
+              if (!known || known.version < deletion.version) {writeVersion(tx, deletion);}
+            });
+          }
         });
       }, reject, resolve);
     });
@@ -177,7 +295,38 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
-  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void> {
+  rejectVersion(operation: PendingMutation, item: Item | null, tombstone: Tombstone | null): Promise<void> {
+    if ((item !== null && item.id !== operation.itemId) || (tombstone !== null && tombstone.id !== operation.itemId)
+        || (item !== null && tombstone !== null)) {
+      return Promise.reject(new Error('Conflict canonical state does not match pending mutation'));
+    }
+    return new Promise((resolve, reject) => {
+      this.database.transaction(tx => readPending(tx, pending => {
+        const first = pending[0];
+        if (!matchesHead(first, operation)) {throw new Error('Conflict does not match the next pending mutation');}
+        // Original envelope, canonical winner and removal from ordinary drains
+        // are one commit. Nothing overwrites this preserved attempt later.
+        writeConflict(tx, {intent: first, reason: 'version_conflict', item, tombstone});
+        tx.executeSql('DELETE FROM pending_mutations WHERE sequence = ?', [first.sequence]);
+        readVersions(tx, versions => {
+          const known = versions.get(first.itemId);
+          const incoming = item !== null ? {...item, deleted: false, item}
+            : tombstone !== null ? {...tombstone, item: null} : null;
+          const canonical = known && (!incoming || !acceptsVersion(known, incoming)) ? known : incoming;
+          tx.executeSql('DELETE FROM items WHERE id = ?', [first.itemId]);
+          if (canonical) {writeVersion(tx, canonical);}
+          else {tx.executeSql('DELETE FROM remote_versions WHERE id = ?', [first.itemId]);}
+          if (canonical?.item) {
+            const row = itemToRow(canonical.item);
+            tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+              [row.id, row.title, row.completed, row.version, row.updated_at]);
+          }
+        });
+      }), reject, resolve);
+    });
+  }
+
+  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number, tombstones: Tombstone[] = []): Promise<void> {
     if (lastSuccessfulRefreshAt !== undefined
         && (!Number.isSafeInteger(lastSuccessfulRefreshAt) || lastSuccessfulRefreshAt < 0)) {
       return Promise.reject(new Error('Invalid successful refresh time'));
@@ -187,17 +336,37 @@ class SqliteItemStore implements ItemStore {
         tx.executeSql('SELECT COUNT(*) AS count FROM pending_mutations', [], (_, result) => {
           // A remote pull cannot erase a local edit that has not uploaded yet.
           if (result.rows.item(0).count !== 0) {throw new Error('Pending uploads must drain before replacing Items');}
-          tx.executeSql('DELETE FROM items');
-          for (const item of items) {
-            const row = itemToRow(item);
-            tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
-              [row.id, row.title, row.completed, row.version, row.updated_at]);
-          }
-          if (lastSuccessfulRefreshAt !== undefined) {
-            // The time describes this exact committed snapshot.
-            tx.executeSql('UPDATE sync_metadata SET last_successful_refresh_at = ? WHERE singleton = 1',
-              [lastSuccessfulRefreshAt]);
-          }
+          readVersions(tx, versions => {
+            const visible = new Map(items.map(item => [item.id, item]));
+            const incoming = new Map<string, RemoteVersion>([
+              ...items.map(item => [item.id, {...item, deleted: false, item}] as const),
+              ...tombstones.map(value => [value.id, {...value, item: null}] as const),
+            ]);
+            for (const value of tombstones) {visible.delete(value.id);}
+            // A late old response (or an unversioned absence) cannot undo a
+            // newer canonical Item/tombstone already committed locally.
+            for (const [id, known] of versions) {
+              const value = incoming.get(id);
+              if (!value || !acceptsVersion(known, value)) {
+                if (known.item && !known.deleted) {visible.set(id, known.item);}
+                else {visible.delete(id);}
+              }
+            }
+            for (const value of incoming.values()) {
+              if (acceptsVersion(versions.get(value.id), value)) {writeVersion(tx, value);}
+            }
+            tx.executeSql('DELETE FROM items');
+            for (const item of visible.values()) {
+              const row = itemToRow(item);
+              tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+                [row.id, row.title, row.completed, row.version, row.updated_at]);
+            }
+            if (lastSuccessfulRefreshAt !== undefined) {
+              // The time describes this exact committed snapshot.
+              tx.executeSql('UPDATE sync_metadata SET last_successful_refresh_at = ? WHERE singleton = 1',
+                [lastSuccessfulRefreshAt]);
+            }
+          });
         });
       }, reject, resolve);
     });
@@ -208,14 +377,14 @@ class SqliteItemStore implements ItemStore {
       let committed: Item[] = [];
       this.database.transaction(tx => {
         readItems(tx, current => {
-          const apply = (identified: ItemAction) => {
+          const apply = (identified: ItemAction, baseVersion = 0) => {
             const next = itemsReducer(current, identified);
             const before = current.find(value => value.id === identified.id);
             const item = next.find(value => value.id === identified.id);
             let operation: UploadOperation | null = null;
             if (identified.type === 'delete' && before) {
               tx.executeSql('DELETE FROM items WHERE id = ?', [identified.id]);
-              operation = {kind: 'delete', itemId: identified.id, payload: null};
+              operation = {kind: 'delete', itemId: identified.id, payload: {baseVersion}};
             } else if (identified.type !== 'delete') {
               if (item && item !== before) {
                 const row = itemToRow(item);
@@ -227,8 +396,8 @@ class SqliteItemStore implements ItemStore {
                   tx.executeSql('UPDATE items SET title = ?, completed = ?, version = ?, updated_at = ? WHERE id = ?',
                     [row.title, row.completed, row.version, row.updated_at, row.id]);
                   operation = identified.type === 'rename'
-                    ? {kind: 'rename', itemId: item.id, payload: {title: item.title}}
-                    : {kind: 'toggle', itemId: item.id, payload: {completed: item.completed}};
+                    ? {kind: 'rename', itemId: item.id, payload: {title: item.title, baseVersion}}
+                    : {kind: 'toggle', itemId: item.id, payload: {completed: item.completed, baseVersion}};
                 }
               }
             }
@@ -236,7 +405,7 @@ class SqliteItemStore implements ItemStore {
               // Store every successful user operation in the same transaction
               // as its Item change (and identity allocation). Never coalesce.
               const identity = identityFor(operation, this.makeIdentity);
-              tx.executeSql('INSERT INTO pending_mutations (kind, item_id, payload, client_mutation_id, payload_hash) VALUES (?, ?, ?, ?, ?)',
+              tx.executeSql('INSERT INTO pending_mutations (kind, item_id, payload, client_mutation_id, payload_hash, dispatched) VALUES (?, ?, ?, ?, ?, 0)',
                 [operation.kind, operation.itemId, operation.payload === null ? null : JSON.stringify(operation.payload),
                   identity.clientMutationId, identity.payloadHash]);
             }
@@ -258,7 +427,12 @@ class SqliteItemStore implements ItemStore {
               apply({...action, id});
             });
           } else if (action.type !== 'create') {
-            apply(action);
+            tx.executeSql('SELECT version FROM remote_versions WHERE id = ?', [action.id], (_, result) => {
+              // Optimistic Item.version is not an observed server version. Zero
+              // means unknown/new local Item; only an own create ACK can prepare
+              // its queued successor, or an explicit refresh can inform a NEW edit.
+              apply(action, result.rows.length ? result.rows.item(0).version : 0);
+            });
           } else {
             committed = current;
           }
@@ -268,13 +442,47 @@ class SqliteItemStore implements ItemStore {
   }
 }
 
+function upgradeFromFour(tx: SQLTransaction) {
+  tx.executeSql("SELECT seq FROM sqlite_sequence WHERE name = 'pending_mutations'", [], (_, sequence) => {
+    const highWater = sequence.rows.length ? sequence.rows.item(0).seq : 0;
+    tx.executeSql('SELECT * FROM pending_mutations ORDER BY sequence', [], (_, saved) => {
+      tx.executeSql('ALTER TABLE pending_mutations RENAME TO pending_mutations_v4');
+      // The old DELETE CHECK required NULL payload. Rebuild just this table so
+      // new deletes carry their hashed base, preserving every original sequence.
+      tx.executeSql("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL CHECK(kind IN ('create', 'rename', 'toggle', 'delete')), item_id TEXT NOT NULL, payload TEXT CHECK(payload IS NOT NULL OR kind = 'delete'), client_mutation_id TEXT NOT NULL, payload_hash TEXT NOT NULL, terminal_error TEXT CHECK(terminal_error IS NULL OR terminal_error = 'identity_conflict'), dispatched INTEGER NOT NULL CHECK(dispatched IN (0, 1)))");
+      tx.executeSql('CREATE TABLE remote_versions (id TEXT PRIMARY KEY NOT NULL, version INTEGER NOT NULL CHECK(version > 0), updated_at INTEGER, deleted INTEGER NOT NULL CHECK(deleted IN (0, 1)), canonical_item TEXT CHECK((deleted = 0 AND canonical_item IS NOT NULL) OR (deleted = 1 AND canonical_item IS NULL)))');
+      tx.executeSql("CREATE TABLE mutation_conflicts (client_mutation_id TEXT PRIMARY KEY NOT NULL, intent TEXT NOT NULL, reason TEXT NOT NULL CHECK(reason IN ('version_conflict', 'unversioned_legacy')), item TEXT, tombstone TEXT)");
+      // Even an empty schema4 queue can follow ACK success + GET failure, with
+      // optimistic Item fields still cached. Leave ALL old canonical provenance
+      // unknown until new remote data is observed; keep those Items readable.
+      for (let i = 0; i < saved.rows.length; i++) {
+        const row = saved.rows.item(i);
+        const intent = decodeMutation({...row, dispatched: 1}, true);
+        if (intent.kind !== 'create' && intent.payload?.baseVersion === undefined && intent.terminalError === null) {
+          // It may already have reached the server. Never invent a base, replace
+          // its identity/hash, or send that unversioned envelope again.
+          writeConflict(tx, {intent, reason: 'unversioned_legacy', item: null, tombstone: null});
+        } else {
+          tx.executeSql('INSERT INTO pending_mutations (sequence, kind, item_id, payload, client_mutation_id, payload_hash, terminal_error, dispatched) VALUES (?, ?, ?, ?, ?, ?, ?, 1)',
+            [row.sequence, row.kind, row.item_id, row.payload, row.client_mutation_id, row.payload_hash, row.terminal_error]);
+        }
+      }
+      tx.executeSql('DROP TABLE pending_mutations_v4');
+      tx.executeSql('CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id)');
+      tx.executeSql("DELETE FROM sqlite_sequence WHERE name = 'pending_mutations'");
+      tx.executeSql("INSERT INTO sqlite_sequence (name, seq) VALUES ('pending_mutations', ?)", [highWater]);
+      tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
+    });
+  });
+}
+
 export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMutationIdentity): Promise<ItemStore> {
   const database = SQLite.openDatabase(name);
   await new Promise<void>((resolve, reject) => {
     database.transaction(tx => {
       tx.executeSql('PRAGMA user_version', [], (_, result) => {
         const version = result.rows.item(0).user_version;
-        if (![0, 1, 2, 3, SCHEMA_VERSION].includes(version)) {
+        if (![0, 1, 2, 3, 4, SCHEMA_VERSION].includes(version)) {
           throw new Error(`Unsupported local database schema ${version}`);
         }
         if (version === 0) {
@@ -302,14 +510,16 @@ export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMuta
           tx.executeSql('ALTER TABLE sync_metadata ADD COLUMN last_acknowledgement TEXT');
           tx.executeSql('SELECT sequence, kind, item_id, payload FROM pending_mutations ORDER BY sequence', [], (_, legacy) => {
             for (let i = 0; i < legacy.rows.length; i++) {
-              const operation = decodeOperation(legacy.rows.item(i));
+              const operation = decodeOperation(legacy.rows.item(i), true);
               const identity = identityFor(operation, makeIdentity);
               tx.executeSql('UPDATE pending_mutations SET client_mutation_id = ?, payload_hash = ? WHERE sequence = ?',
                 [identity.clientMutationId, identity.payloadHash, operation.sequence]);
             }
             tx.executeSql('CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id)');
-            tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
+            upgradeFromFour(tx);
           });
+        } else if (version === 4) {
+          upgradeFromFour(tx);
         }
       });
     }, reject, resolve);
diff --git a/src/sync.ts b/src/sync.ts
index fde8e37..3188251 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -1,5 +1,5 @@
 import {Item} from './items';
-import {ItemStore, MutationResult, PendingMutation} from './itemStore';
+import {ItemStore, MutationResult, PendingMutation, Tombstone} from './itemStore';
 import {mutationTarget} from './mutationProtocol';
 
 export const FIXTURE_URL = 'http://10.0.2.2:18081';
@@ -27,6 +27,14 @@ function remoteItem(value: unknown): Item {
     version: item.version, updatedAt: item.updatedAt};
 }
 
+function remoteTombstone(value: unknown): Tombstone {
+  const tombstone = value as Tombstone;
+  if (!tombstone || typeof tombstone.id !== 'string' || !tombstone.id || tombstone.deleted !== true
+      || !Number.isSafeInteger(tombstone.version) || tombstone.version < 1
+      || !Number.isSafeInteger(tombstone.updatedAt)) {throw new Error('Invalid remote tombstone');}
+  return {id: tombstone.id, version: tombstone.version, updatedAt: tombstone.updatedAt, deleted: true};
+}
+
 // Item IDs need distinct namespaces as soon as devices share remote state.
 // Full IDs persist in SQLite. This nonce is not a clientMutationId protocol.
 function sessionIdentity() {
@@ -51,11 +59,17 @@ export class ForegroundSync implements SyncSession {
     });
     if (response.status !== expectedStatus) {
       if (operation && response.status === 409) {
-        const failure = await response.json() as {error?: string};
+        const failure = await response.json() as {error?: string; item?: unknown; tombstone?: unknown};
         if (failure?.error === 'identity_conflict') {
           await this.store.rejectIdentity(operation);
           throw new Error('Mutation identity conflict; upload stopped without retry');
         }
+        if (failure?.error === 'version_conflict' && operation.kind !== 'create') {
+          const item = failure.item === null ? null : remoteItem(failure.item);
+          const tombstone = failure.tombstone === null ? null : remoteTombstone(failure.tombstone);
+          await this.store.rejectVersion(operation, item, tombstone);
+          return null;
+        }
       }
       throw new Error(`${method} ${path} failed (HTTP ${response.status})`);
     }
@@ -65,14 +79,16 @@ export class ForegroundSync implements SyncSession {
   async synchronize(): Promise<void> {
     // Recover ordered upload intent from SQLite, including on the first sync
     // after process restart. Each edit is sent separately, without coalescing.
-    const pending = await this.store.readPending();
-    for (const operation of pending) {
+    for (;;) {
+      const operation = await this.store.prepareNext();
+      if (operation === null) {break;}
       if (operation.terminalError === 'identity_conflict') {
         throw new Error('Mutation identity conflict; upload stopped without retry');
       }
       const target = mutationTarget(operation.kind, operation.itemId);
       const response = await this.exchange(target.method, target.path, target.status,
-        {...operation.payload, clientMutationId: operation.clientMutationId, payloadHash: operation.payloadHash}, operation) as {item?: unknown; deletedId?: unknown};
+        {...operation.payload, clientMutationId: operation.clientMutationId, payloadHash: operation.payloadHash}, operation) as {item?: unknown; deletedId?: unknown} | null;
+      if (response === null) {continue;} // Preserved conflict is no longer pending.
       let result: MutationResult;
       if (operation.kind === 'delete') {
         if (response?.deletedId !== operation.itemId) {throw new Error('Invalid remote deletion acknowledgment');}
@@ -85,13 +101,16 @@ export class ForegroundSync implements SyncSession {
       await this.store.acknowledge(operation, result);
     }
 
-    const response = await this.exchange('GET', '/items', 200) as {items?: unknown};
-    if (!Array.isArray(response.items)) {throw new Error('Invalid remote snapshot');}
+    const response = await this.exchange('GET', '/items', 200) as {items?: unknown; tombstones?: unknown};
+    if (!Array.isArray(response.items) || (response.tombstones !== undefined && !Array.isArray(response.tombstones))) {
+      throw new Error('Invalid remote snapshot');
+    }
     const snapshot = response.items.map(remoteItem);
-    if (new Set(snapshot.map(item => item.id)).size !== snapshot.length) {
-      throw new Error('Duplicate Item in remote snapshot');
+    const tombstones = ((response.tombstones ?? []) as unknown[]).map(remoteTombstone);
+    if (new Set([...snapshot, ...tombstones].map(item => item.id)).size !== snapshot.length + tombstones.length) {
+      throw new Error('Duplicate Item or tombstone in remote snapshot');
     }
-    await this.store.replaceSnapshot(snapshot, this.now());
+    await this.store.replaceSnapshot(snapshot, this.now(), tombstones);
     this.refreshed = true;
   }
 }
diff --git a/verification/M07.md b/verification/M07.md
index d76e42e..3589e02 100644
--- a/verification/M07.md
+++ b/verification/M07.md
@@ -33,3 +33,23 @@ Frozen `repair1-baseline-C-01` (`--baseline --case C`) exited0 in 49.932s. The e
 The reset gate passed in 1.137s with seven recorded absent-PID/package observations. It ran successfully, but this C-only execution did not deliberately recreate the old teardown race. Cleanup restored network `0/1/1`, left app PID absent and reaped owned fixture70992 with exit0. No additional device execution or APK rebuild occurred.
 
 Main independently accepted the actual C database/UI/wire evidence and frozen byte identity; audit: `threads/evidence/phase-1/react-native/M07/main-repair1-C-audit.json` in the main worktree. Original A/B evidence plus this C run complete baseline reproduction only. Repair usage stays **1/2**; M07 product implementation and official final A/B/C verification remain outstanding. No M07 completion tag is created by this repair.
+
+## Product implementation after baseline completion
+
+Cumulative attempt2, repair1/2 used; START and both existing M07 commits remain intact. Schema5 adds hashed bases, durable dispatch, canonical observations and original conflict evidence. Own-ACK successor preparation shares the ACK/dequeue commit; external versions never rebase existing intent. Unknown schema4 provenance stays unknown until refresh. Canonical payloads and an untimestamped DELETE receipt marker prevent older responses from publishing optimistic fields or resurrecting deletion. The UI explains canonical-wins and explicit new edits; no new dependency or framework.
+
+`host-typecheck-01` PASS; `host-jest-01` **75/75 PASS**; `host-harness-syntax-01` PASS. Exact argv, source snapshots, outputs and exits are in the evidence root (`*.command.json`, `*.log`, `snapshots/`). Coverage retains M01–M06, including four M05 accepts and exact M06 replay, and adds A/B/C+explicit edit, original conflict persistence/no-send, own-successor loss/replay, external-version rejection, base-only collision, transaction rollback and both schema4 provenance cases. Older harnesses advance only current schema assertions and required version/tombstone fields; original scenario inputs and domain results are unchanged. The repaired M07 harness, fixture and M07 inputs remain frozen.
+
+Android product verification is **NOT_RUN** until main tests the frozen candidate; no owner final device run is authorized.
+
+`android-build-01` PASS (22.282s): debug app and existing test runner. Preserved artifacts/hashes are in `artifacts.json`; the app is `m07-candidate.apk` (SHA256 `a42cc4c143e51aae73ff3645d829f2491e457c29607c659f2852c45d483391f7`), and the test-runner bytes remain unchanged from M06. `candidate-manifest.json` freezes all current files, host/build snapshots, artifacts and prior raw evidence paths.
+
+Main-only final invocation, from `/private/tmp/mobile-systems-evolution-ed7baa2/react-native`, after an exclusive device lease and a free fixture port18081:
+
+```sh
+python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/run.py main-android-m07-01 python3 scripts/verify_m07.py \
+  --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 \
+  --node /Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin/node \
+  --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/m07-candidate.apk \
+  --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M07/main-android-m07-01
+```


