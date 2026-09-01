## `feat(M06): persist mutation identity and atomically acknowledge replay`

diff --git a/TRACK.md b/TRACK.md
index 83dea7e..348c8bf 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -109,6 +109,36 @@ module is added. `scripts/verify_m05.py` freezes the actual offline four-edit,
 process-kill/relaunch and one-drain scenario; see `verification/M05.md` for hashes,
 raw failures and the coordinator's final Android verification.
 
+## M06: durable identity and lost-acknowledgment recovery (phase-1)
+
+Schema v4 adds `client_mutation_id`, `payload_hash` and a terminal identity-error
+marker to each queued operation. A fresh mutation nonce, its canonical hash, the
+Item edit and intent commit together. Startup and every drain validate persisted
+payload hashes. Legacy queue rows receive identity/hash in one migration commit;
+their original sequence and allocator high-water marks remain intact. An earlier
+M05 request without identity cannot retroactively gain remote duplicate evidence.
+
+Every mutation sends the two protocol fields outside its domain payload. The hash
+is SHA-256 of compact UTF-8 JSON `{method,path,payload}`, with object keys sorted at
+every level. Pinned `js-sha256`0.11.1 supplies the JavaScript hash implementation;
+the fixture independently validates actual bodies with Node crypto. There is no
+native hashing module. The Android debug launch override selects only the fixed
+Item/mutation identifiers, leaving production SQLite and fetch in use.
+
+After a validated response, the last acknowledged identity/hash/status/result and
+dequeue share one native transaction. Recording the result avoids overwriting a
+later optimistic edit with an earlier reply; the final successful GET still
+commits canonical Items and refresh time. Any failed acknowledgment commit leaves
+the intent available for an identical replay. The fixture returns its original
+status/body for a duplicate without advancing version, timestamp or side effects.
+
+A valid different payload under the same identity receives409 `identity_conflict`.
+The full local attempt remains stored with that terminal marker, which is visible
+after restart and never automatically resent. No stale-version policy, scheduler,
+backoff, push, state framework or extra business feature is introduced. The frozen
+M06 Android harness covers lost response, actual process death/replay, a separate
+production-created collision and wrong-hash rejection. See `verification/M06.md`.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 04d3f02..492de4d 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -256,7 +256,8 @@ test('M05 UI reads durable pending work after restart and clears it after a fore
     const reply = replies.shift();
     expect(reply).toBeDefined();
     expect(options.method).toBe(reply!.method);
-    expect(options.body ? JSON.parse(options.body) : null).toEqual(reply!.body);
+    expect(options.body ? JSON.parse(options.body) : null).toEqual(reply!.method === 'GET' ? reply!.body
+      : {...reply!.body, clientMutationId: expect.any(String), payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/)});
     return {status: reply!.status, json: async () => reply!.response};
   });
   const store = await openItemStore();
@@ -317,3 +318,23 @@ test('M05 UI reads durable pending work after restart and clears it after a fore
     expect(request).toHaveBeenCalledTimes(5);
   } finally {clock.mockRestore();}
 });
+
+test('M06 startup exposes a persisted identity collision and an explicit drain cannot resend it', async () => {
+  const store = await openItemStore(undefined, () => 'm06-create-001');
+  await store.mutate({type: 'create', title: 'Different payload', now: 1700000399000}, 'crash');
+  await store.rejectIdentity((await store.readPending())[0]);
+  closeDatabases();
+  const reopened = await openItemStore();
+  const request: JsonRequest = jest.fn(() => Promise.reject(new Error('Terminal intent must not send')));
+  render(<App openStore={async () => reopened}
+    createSync={() => new ForegroundSync(reopened, 'http://fixed-m06', request)} />);
+  await saved();
+  expect(screen.getByText('Different payload')).toBeTruthy();
+  expect(screen.getByLabelText('Pending uploads: 1')).toBeTruthy();
+  expect(screen.getByText('Upload blocked: mutation identity conflict. It will not be retried.')).toBeTruthy();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await waitFor(() => expect(screen.getByLabelText('Sync status: error')).toBeTruthy());
+  await saved();
+  expect(request).not.toHaveBeenCalled();
+  expect((await reopened.readPending())[0].terminalError).toBe('identity_conflict');
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index 384440a..9ec5b8c 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -2,6 +2,14 @@ import {Item, itemsReducer} from '../src/items';
 import SQLite from 'react-native-sqlite-2';
 import {ItemMutation, itemToRow, openItemStore, PendingMutation, rowToItem} from '../src/itemStore';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
+import {canonicalJson, mutationHash, mutationTarget} from '../src/mutationProtocol';
+
+const m06 = require('../verification/M06-inputs.json') as {
+  clientMutationId: string; payloadHash: string; baselineLocalTimestamp: number;
+  payload: {id: string; title: string; completed: boolean}; canonicalItem: Item;
+  acknowledgement: unknown;
+  hashVectors: {method: string; path: string; payload: unknown; canonical: string; sha256: string}[];
+};
 
 test('M01 fixed sequence preserves first identity and all Item fields', () => {
   let items: Item[] = [];
@@ -82,11 +90,11 @@ test.each([/^INSERT INTO items/, /^END/])('M02 failure at %s rolls back Item and
 test('M02 unsupported schema is rejected without recreating or deleting existing data', async () => {
   const store = await openItemStore();
   const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  connection().exec('PRAGMA user_version = 4');
+  connection().exec('PRAGMA user_version = 5');
   closeDatabases();
-  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 4');
+  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 5');
   expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(4);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(5);
 });
 
 test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
@@ -125,7 +133,7 @@ test('M04 upgrades a literal M03 schema without touching cached Items or local i
   const store = await openItemStore(name);
   expect(await store.read()).toEqual(seeds);
   expect(await store.readLastSuccessfulRefresh()).toBeNull();
-  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(3);
+  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(4);
   expect(database.prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
   closeDatabases();
   const reopened = await openItemStore(name);
@@ -144,12 +152,13 @@ const m05Actions: ItemMutation[] = [
   {type: 'toggle', id: 'remote-001', now: 1700000302000},
   {type: 'delete', id: 'remote-002'},
 ];
-const m05Pending: PendingMutation[] = [
+const m05Pending = [
   {sequence: 1, kind: 'create', itemId: 'device-001', payload: {id: 'device-001', title: 'Gamma', completed: false}},
   {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
   {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
   {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
-];
+].map(operation => ({...operation, clientMutationId: expect.any(String),
+  payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null}));
 
 const mutationFailures = [
   {index: 0, kind: 'create', localSql: /^INSERT INTO items/},
@@ -175,7 +184,13 @@ test.each(mutationFailures)('M05 $kind rollback at $failAt preserves both Item a
   expect(await reopened.readPending()).toEqual(beforePending);
   expect(connection().prepare('SELECT next_id FROM local_identity WHERE singleton=1').get()?.next_id).toBe(identity);
   await reopened.mutate(m05Actions[index], 'device');
-  expect(await reopened.readPending()).toEqual(m05Pending.slice(0, index + 1));
+  const afterRetry = await reopened.readPending();
+  expect(afterRetry).toEqual(m05Pending.slice(0, index + 1));
+  expect(new Set(afterRetry.map(operation => operation.clientMutationId)).size).toBe(afterRetry.length);
+  for (const operation of afterRetry) {
+    const target = mutationTarget(operation.kind, operation.itemId);
+    expect(operation.payloadHash).toBe(mutationHash(target.method, target.path, operation.payload));
+  }
   console.info('M05_ATOMIC_ROLLBACK', JSON.stringify({operation: m05Actions[index], failure: String(failAt),
     before, beforePending, afterRetry: await reopened.read(), pendingAfterRetry: await reopened.readPending()}));
 });
@@ -236,5 +251,115 @@ test('M05 v2 migration preserves Items, allocator and refresh time, including fa
   expect(await reopened.read()).toEqual(m05Seeds);
   expect(await reopened.readPending()).toEqual([]);
   expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(3);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(4);
+});
+
+test.each(m06.hashVectors)('M06 shared compact UTF-8 canonical vector $method $path $sha256', vector => {
+  expect(canonicalJson({method: vector.method, path: vector.path, payload: vector.payload})).toBe(vector.canonical);
+  expect(mutationHash(vector.method, vector.path, vector.payload)).toBe(vector.sha256);
+  // The fixture inputs include nested objects, numeric-looking keys, arrays,
+  // Unicode values and standard JSON escaping; the oracle was Python/hashlib.
+  const reordered = (value: unknown): unknown => Array.isArray(value) ? value.map(reordered)
+    : value && typeof value === 'object'
+      ? Object.fromEntries(Object.keys(value).reverse().map(key => [key, reordered((value as Record<string, unknown>)[key])])) : value;
+  expect(mutationHash(vector.method, vector.path, reordered(vector.payload))).toBe(vector.sha256);
+});
+
+const m06Pending: PendingMutation = {sequence: 1, kind: 'create', itemId: 'crash-001', payload: m06.payload,
+  clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null};
+const lastAcknowledgement = () => {
+  const value = connection().prepare('SELECT last_acknowledgement FROM sync_metadata WHERE singleton=1').get()?.last_acknowledgement;
+  return value === null ? null : JSON.parse(String(value));
+};
+
+test.each([/^UPDATE sync_metadata SET last_acknowledgement/, /^DELETE FROM pending_mutations/, /^END/])(
+  'M06 acknowledgment failure at %s rolls back result recording and dequeue together', async failAt => {
+    const store = await openItemStore(undefined, () => m06.clientMutationId);
+    const local = await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
+    expect(await store.readPending()).toEqual([m06Pending]);
+    expect(lastAcknowledgement()).toBeNull();
+    failNextSql(failAt);
+    await expect(store.acknowledge(m06Pending, {item: m06.canonicalItem})).rejects.toThrow('injected persistence failure');
+    closeDatabases();
+    const reopened = await openItemStore();
+    expect(await reopened.readPending()).toEqual([m06Pending]);
+    expect(await reopened.read()).toEqual(local);
+    expect(lastAcknowledgement()).toBeNull();
+    await reopened.acknowledge(m06Pending, {item: m06.canonicalItem});
+    expect(await reopened.readPending()).toEqual([]);
+    expect(lastAcknowledgement()).toEqual(m06.acknowledgement);
+    closeDatabases();
+    expect(await (await openItemStore()).readPending()).toEqual([]);
+    expect(lastAcknowledgement()).toEqual(m06.acknowledgement);
+    console.info('M06_ATOMIC_ACK', JSON.stringify({failure: String(failAt), acknowledgement: lastAcknowledgement()}));
+  });
+
+test('M06 foreign, out-of-order or terminal acknowledgments cannot erase intent', async () => {
+  let next = 0;
+  const store = await openItemStore(undefined, () => `check-${++next}`);
+  await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
+  await store.mutate({type: 'rename', id: 'crash-001', title: 'Still local', now: 1700000401000});
+  const pending = await store.readPending();
+  await expect(store.acknowledge(pending[0], {item: {...m06.canonicalItem, id: 'another'}})).rejects.toThrow('does not match');
+  await expect(store.acknowledge({...pending[0], clientMutationId: 'another'}, {item: m06.canonicalItem})).rejects.toThrow('does not match');
+  await expect(store.acknowledge(pending[1], {item: m06.canonicalItem})).rejects.toThrow('does not match');
+  expect(await store.readPending()).toEqual(pending);
+  expect(lastAcknowledgement()).toBeNull();
+  await store.rejectIdentity(pending[0]);
+  await expect(store.acknowledge(pending[0], {item: m06.canonicalItem})).rejects.toThrow('does not match');
+  expect((await store.readPending())[0]).toEqual({...pending[0], terminalError: 'identity_conflict'});
+  expect(lastAcknowledgement()).toBeNull();
+});
+
+test('M06 accepting an earlier intent records its result without overwriting a later local edit', async () => {
+  let next = 0;
+  const store = await openItemStore(undefined, () => `ordered-${++next}`);
+  await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
+  const edited = await store.mutate({type: 'rename', id: 'crash-001', title: 'Still local', now: 1700000401000});
+  const pending = await store.readPending();
+  await store.acknowledge(pending[0], {item: m06.canonicalItem});
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(edited);
+  expect(await reopened.readPending()).toEqual([pending[1]]);
+  expect(lastAcknowledgement()).toEqual({...m06.acknowledgement as object, clientMutationId: pending[0].clientMutationId});
 });
+
+test.each([/^UPDATE pending_mutations SET client_mutation_id/, /^CREATE UNIQUE INDEX/, /^END/])(
+  'M06 schema3 migration failure at %s preserves legacy intent and assigns durable identity only on commit', async failAt => {
+    const database = connection();
+    database.exec(`
+      CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL, completed INTEGER NOT NULL, version INTEGER NOT NULL, updated_at INTEGER NOT NULL);
+      CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY, next_id INTEGER NOT NULL);
+      INSERT INTO local_identity VALUES (1, 2);
+      CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY, last_successful_refresh_at INTEGER);
+      INSERT INTO sync_metadata VALUES (1, 1700000200000);
+      CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL, item_id TEXT NOT NULL, payload TEXT);
+      PRAGMA user_version = 3;
+    `);
+    database.prepare('INSERT INTO items VALUES (?, ?, ?, ?, ?)').run('crash-001', 'Crash safe', 0, 1, m06.baselineLocalTimestamp);
+    database.prepare('INSERT INTO pending_mutations VALUES (?, ?, ?, ?)').run(7, 'create', 'crash-001', JSON.stringify(m06.payload));
+    database.exec("UPDATE sqlite_sequence SET seq=9 WHERE name='pending_mutations'");
+    const local = database.prepare('SELECT * FROM items').all();
+    const legacy = database.prepare('SELECT * FROM pending_mutations').all();
+    failNextSql(failAt);
+    await expect(openItemStore(undefined, () => m06.clientMutationId)).rejects.toThrow('injected persistence failure');
+    expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(3);
+    expect(database.prepare('SELECT * FROM pending_mutations').all()).toEqual(legacy);
+    expect(database.prepare('SELECT * FROM items').all()).toEqual(local);
+    expect(database.prepare('SELECT * FROM sync_metadata').get()).toEqual({singleton: 1, last_successful_refresh_at: 1700000200000});
+    const migrated = await openItemStore(undefined, () => m06.clientMutationId);
+    expect(await migrated.readPending()).toEqual([{...m06Pending, sequence: 7}]);
+    expect(await migrated.readLastSuccessfulRefresh()).toBe(1700000200000);
+    expect(database.prepare('SELECT * FROM items').all()).toEqual(local);
+    closeDatabases();
+    const makeIdentity = jest.fn(() => 'new-after-upgrade');
+    const reopened = await openItemStore(undefined, makeIdentity);
+    expect(await reopened.readPending()).toEqual([{...m06Pending, sequence: 7}]);
+    expect(makeIdentity).not.toHaveBeenCalled();
+    await reopened.mutate({type: 'create', title: 'Next', now: 1700000401000}, 'crash');
+    const next = (await reopened.readPending())[1];
+    expect(next.sequence).toBe(10);
+    expect(next.itemId).toBe('crash-002');
+    expect(next.clientMutationId).toBe('new-after-upgrade');
+  });
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 0c0dc28..366ad83 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -2,11 +2,12 @@
 import {request as httpRequest, Server} from 'node:http';
 import {ForegroundSync, JsonRequest} from '../src/sync';
 import {openItemStore} from '../src/itemStore';
-import {closeDatabases, failNextSql} from './sqliteNative';
+import {closeDatabases, connection, failNextSql} from './sqliteNative';
 
 type Trace = {method: string; path: string; body: unknown; status: number; response: unknown};
 const {createFixture} = require('../fixture/server.cjs') as {createFixture(): {
   server: Server; reset(): void; state(): {items: unknown[]; nextTimestamp: number; requests: Trace[]};
+  mutationState(): {appliedCount: number; duplicateCount: number; conflictCount: number; hashRejectedCount: number; attempts: unknown[]};
 }};
 const fixture = createFixture();
 const url = 'http://127.0.0.1:18081';
@@ -14,9 +15,15 @@ const url = 'http://127.0.0.1:18081';
 // The service uses RN fetch on Android. On the host this transport speaks real
 // loopback HTTP instead of Jest's RN XMLHttpRequest mock; it does not mock replies.
 const request: JsonRequest = (address, options) => new Promise((resolve, reject) => {
-  const outgoing = httpRequest(address, options, response => {
+  // Node does not automatically frame a DELETE body; RN fetch does. Preserve
+  // the same JSON bytes, with their UTF-8 length, in this host-only adapter.
+  const headers = {...options.headers,
+    ...(options.body === undefined ? {} : {'Content-Length': String(Buffer.byteLength(options.body, 'utf8'))})};
+  const outgoing = httpRequest(address, {...options, headers}, response => {
     const chunks: Buffer[] = [];
     response.on('data', chunk => chunks.push(chunk));
+    response.on('aborted', () => reject(new Error('Response closed before full body')));
+    response.on('error', reject);
     response.on('end', () => resolve({status: response.statusCode!,
       json: async () => JSON.parse(Buffer.concat(chunks).toString('utf8'))}));
   });
@@ -257,7 +264,8 @@ test('M05 one failed offline foreground drain retains every ordered intent for r
     {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
     {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
     {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
-  ]);
+  ].map(operation => ({...operation, clientMutationId: expect.any(String),
+    payloadHash: expect.stringMatching(/^[a-f0-9]{64}$/), terminalError: null})));
   await expect(sync.synchronize()).rejects.toThrow('Network request failed');
   expect(fixture.state().requests).toHaveLength(1);
   expect(fixture.state().items).toEqual(seeds);
@@ -266,3 +274,81 @@ test('M05 one failed offline foreground drain retains every ordered intent for r
   expect(await reopened.read()).toEqual(local);
   expect(await reopened.readPending()).toEqual(pending);
 });
+
+const m06 = require('../verification/M06-inputs.json');
+
+test('M06 lost acknowledgment resends the exact durable identity and returns the original result once', async () => {
+  await control('/__m06-reset', {});
+  const store = await openItemStore(undefined, () => m06.clientMutationId);
+  const local = await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
+  const pending = await store.readPending();
+  expect(pending).toEqual([{sequence: 1, kind: 'create', itemId: 'crash-001', payload: m06.payload,
+    clientMutationId: m06.clientMutationId, payloadHash: m06.payloadHash, terminalError: null}]);
+  await expect(new ForegroundSync(store, url, request).synchronize()).rejects.toThrow('Response closed before full body');
+  expect(await store.readPending()).toEqual(pending);
+  expect(await store.read()).toEqual(local);
+  expect(fixture.mutationState().appliedCount).toBe(1);
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.readPending()).toEqual(pending);
+  expect(await reopened.read()).toEqual(local);
+  await new ForegroundSync(reopened, url, request).synchronize();
+  expect(await reopened.read()).toEqual([m06.canonicalItem]);
+  expect(await reopened.readPending()).toEqual([]);
+  expect(JSON.parse(String(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement))).toEqual(m06.acknowledgement);
+  const attempt = {method: 'POST', path: '/items', wireBody: m06.wireBody,
+    clientMutationId: m06.clientMutationId, declaredHash: m06.payloadHash, actualHash: m06.payloadHash,
+    canonical: m06.canonicalJson, status: 201, response: {item: m06.canonicalItem}};
+  expect(fixture.mutationState()).toEqual({appliedCount: 1, duplicateCount: 1, conflictCount: 0, hashRejectedCount: 0, attempts: [
+    {...attempt, outcome: 'applied', responseDropped: true}, {...attempt, outcome: 'duplicate', responseDropped: false},
+  ]});
+  expect(fixture.state()).toEqual({items: [m06.canonicalItem], nextTimestamp: 1700000401000, requests: [
+    {method: 'POST', path: '/items', body: m06.payload, status: 201, response: {item: m06.canonicalItem}},
+    {method: 'POST', path: '/items', body: m06.payload, status: 201, response: {item: m06.canonicalItem}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [m06.canonicalItem]}},
+  ]});
+  console.info('M06_LOST_ACK_REPLAY', JSON.stringify({pendingBefore: pending, remote: fixture.state(), evidence: fixture.mutationState()}));
+});
+
+test('M06 a valid identity collision becomes durable terminal intent; a wrong hash is rejected against the actual body', async () => {
+  await control('/__m06-reset', {});
+  const wire = (body: unknown) => request(url + '/items', {method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify(body)});
+  await expect(wire(m06.wireBody)).rejects.toThrow('Response closed before full body');
+  const store = await openItemStore(undefined, () => m06.clientMutationId);
+  const local = await store.mutate({type: 'create', title: 'Different payload', now: m06.baselineLocalTimestamp}, 'crash');
+  const pending = await store.readPending();
+  expect(pending[0].payloadHash).toBe(m06.collisionHash);
+  await expect(new ForegroundSync(store, url, request).synchronize()).rejects.toThrow('Mutation identity conflict');
+  const terminal = [{...pending[0], terminalError: 'identity_conflict'}];
+  expect(await store.readPending()).toEqual(terminal);
+  const collision = fixture.mutationState();
+  expect(collision.conflictCount).toBe(1);
+  expect(collision.attempts[1]).toMatchObject({wireBody: m06.collisionWireBody, status: 409, response: {error: 'identity_conflict'}});
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.readPending()).toEqual(terminal);
+  expect(await reopened.read()).toEqual(local);
+  await expect(new ForegroundSync(reopened, url, request).synchronize()).rejects.toThrow('Mutation identity conflict');
+  expect(fixture.mutationState()).toEqual(collision);
+  expect(fixture.state().requests).toHaveLength(2);
+  const invalid = await wire(m06.wrongHashWireBody);
+  expect(invalid.status).toBe(400);
+  expect(await invalid.json()).toEqual({error: 'payload_hash_mismatch'});
+  expect(fixture.mutationState().attempts[2]).toMatchObject({declaredHash: m06.payloadHash, actualHash: m06.collisionHash,
+    outcome: 'hash_mismatch', status: 400});
+  expect(fixture.mutationState()).toMatchObject({appliedCount: 1, duplicateCount: 0, conflictCount: 1, hashRejectedCount: 1});
+  expect(fixture.state().items).toEqual([m06.canonicalItem]);
+  expect(fixture.state().nextTimestamp).toBe(1700000401000);
+  console.info('M06_TERMINAL_COLLISION', JSON.stringify({terminal, remote: fixture.state(), evidence: fixture.mutationState()}));
+});
+
+test('M06 an invalid remote acknowledgment never silently removes durable intent', async () => {
+  const store = await openItemStore(undefined, () => m06.clientMutationId);
+  await store.mutate({type: 'create', title: 'Crash safe', now: m06.baselineLocalTimestamp}, 'crash');
+  const pending = await store.readPending();
+  const invalid: JsonRequest = async () => ({status: 201, json: async () => ({item: {...m06.canonicalItem, id: 'wrong'}})});
+  await expect(new ForegroundSync(store, url, invalid).synchronize()).rejects.toThrow('belongs to another Item');
+  closeDatabases();
+  expect(await (await openItemStore()).readPending()).toEqual(pending);
+  expect(connection().prepare('SELECT last_acknowledgement FROM sync_metadata').get()?.last_acknowledgement).toBeNull();
+});
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index 812ad57..3dfe55a 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -18,6 +18,10 @@ class MainActivity : ReactActivity() {
                     if (intent.getBooleanExtra("m04FixedClock", false)) {
                         putBoolean("testRefreshClock", true)
                     }
+                    if (intent.getBooleanExtra("m06FixedIdentity", false)) {
+                        putString("testIdentityPrefix", "crash")
+                        putString("testMutationIdentity", "m06-create-001")
+                    }
                 }
             }
         }
diff --git a/package-lock.json b/package-lock.json
index 14f9486..23261bf 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -7,7 +7,9 @@
     "": {
       "name": "offline-item-tracker",
       "version": "0.1.0",
+      "hasInstallScript": true,
       "dependencies": {
+        "js-sha256": "0.11.1",
         "react": "18.3.1",
         "react-native": "0.76.9",
         "react-native-sqlite-2": "3.6.3"
@@ -7340,6 +7342,12 @@
         "@sideway/pinpoint": "^2.0.0"
       }
     },
+    "node_modules/js-sha256": {
+      "version": "0.11.1",
+      "resolved": "https://registry.npmjs.org/js-sha256/-/js-sha256-0.11.1.tgz",
+      "integrity": "sha512-o6WSo/LUvY2uC4j7mO50a2ms7E/EAdbP0swigLV+nzHKTTaYnaLIWJ02VdXrsJX0vGedDESQnLsOekr94ryfjg==",
+      "license": "MIT"
+    },
     "node_modules/js-tokens": {
       "version": "4.0.0",
       "resolved": "https://registry.npmjs.org/js-tokens/-/js-tokens-4.0.0.tgz",
diff --git a/package.json b/package.json
index eb00a2c..f3f337f 100644
--- a/package.json
+++ b/package.json
@@ -8,6 +8,7 @@
     "typecheck": "tsc --noEmit"
   },
   "dependencies": {
+    "js-sha256": "0.11.1",
     "react": "18.3.1",
     "react-native": "0.76.9",
     "react-native-sqlite-2": "3.6.3"
diff --git a/scripts/verify_m02.py b/scripts/verify_m02.py
index 214d619..b5c8e24 100644
--- a/scripts/verify_m02.py
+++ b/scripts/verify_m02.py
@@ -99,11 +99,11 @@ def main():
         assert path.exists(), "Native database file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as database:
             assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert database.execute("PRAGMA user_version").fetchone()[0] == 3
+            assert database.execute("PRAGMA user_version").fetchone()[0] == 4
             assert [column[1] for column in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in database.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid")]
-            state = {"schema_version": 3, "items": items,
+            state = {"schema_version": 4, "items": items,
                      "next_id": database.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(state, indent=2))
         return state
diff --git a/scripts/verify_m03.py b/scripts/verify_m03.py
index 9f1b906..b73db64 100644
--- a/scripts/verify_m03.py
+++ b/scripts/verify_m03.py
@@ -138,11 +138,11 @@ def main():
         assert path.exists(), "Native SQLite file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert db.execute("PRAGMA user_version").fetchone()[0] == 3
+            assert db.execute("PRAGMA user_version").fetchone()[0] == 4
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
-            value = {"items": items, "schema_version": 3,
+            value = {"items": items, "schema_version": 4,
                      "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
         return value
diff --git a/scripts/verify_m04.py b/scripts/verify_m04.py
index 80eea27..b0810b2 100644
--- a/scripts/verify_m04.py
+++ b/scripts/verify_m04.py
@@ -116,7 +116,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (1 if args.baseline else 3), schema
+            assert schema == (1 if args.baseline else 4), schema
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
diff --git a/scripts/verify_m05.py b/scripts/verify_m05.py
index ccf8996..50dab68 100644
--- a/scripts/verify_m05.py
+++ b/scripts/verify_m05.py
@@ -147,7 +147,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (2 if args.baseline else 3), schema
+            assert schema == (2 if args.baseline else 4), schema
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
             tables = [row[0] for row in db.execute("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
diff --git a/src/App.tsx b/src/App.tsx
index 9ceb61e..9d14ffc 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -14,10 +14,11 @@ const defaultSync = (store: ItemStore, identityPrefix?: string, testRefreshClock
 
 type RefreshState = {status: 'stale' | 'refreshing' | 'fresh'} | {status: 'error'; message: string};
 
-export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix, testRefreshClock = false}: {
+export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix, testMutationIdentity, testRefreshClock = false}: {
   openStore?: () => Promise<ItemStore>;
   createSync?: (store: ItemStore, identityPrefix?: string, testRefreshClock?: boolean) => SyncSession;
   testIdentityPrefix?: string;
+  testMutationIdentity?: string;
   testRefreshClock?: boolean;
 }) {
   const [items, setItems] = useState<Item[]>([]);
@@ -27,6 +28,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const [refresh, setRefresh] = useState<RefreshState>({status: 'stale'});
   const [lastSuccessfulRefreshAt, setLastSuccessfulRefreshAt] = useState<number | null>(null);
   const [pendingCount, setPendingCount] = useState<number | null>(null);
+  const [identityBlocked, setIdentityBlocked] = useState(false);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
@@ -39,7 +41,10 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     busyRef.current = true;
     setBusy(true);
     setError(null);
-    openStore().then(async opened => {
+    // The Android override is debug-only and changes just the generated value.
+    const opening = openStore === openItemStore && testMutationIdentity
+      ? openItemStore(undefined, () => testMutationIdentity) : openStore();
+    opening.then(async opened => {
       const saved = await opened.read();
       const lastRefresh = await opened.readLastSuccessfulRefresh();
       const pending = await opened.readPending();
@@ -49,6 +54,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         setItems(saved);
         setLastSuccessfulRefreshAt(lastRefresh);
         setPendingCount(pending.length);
+        setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
         setRefresh({status: 'stale'});
         setReady(true);
       }
@@ -58,10 +64,14 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       if (active) {busyRef.current = false; setBusy(false);}
     });
     return () => {active = false;};
-  }, [openStore, openAttempt, createSync, testIdentityPrefix, testRefreshClock]);
+  }, [openStore, openAttempt, createSync, testIdentityPrefix, testMutationIdentity, testRefreshClock]);
 
   async function reloadPending() {
-    try {setPendingCount((await store.current!.readPending()).length);}
+    try {
+      const pending = await store.current!.readPending();
+      setPendingCount(pending.length);
+      setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
+    }
     catch {setPendingCount(null);} // A failed status read must not claim a committed edit was unsaved.
   }
 
@@ -142,6 +152,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         <Text accessibilityLabel={`Pending uploads: ${pendingCount ?? 'unknown'}`}>
           {pendingCount === null ? 'Pending upload count unavailable' : `Pending uploads: ${pendingCount}`}
         </Text>
+        {identityBlocked && <Text accessibilityRole="alert">Upload blocked: mutation identity conflict. It will not be retried.</Text>}
         {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
       </>}
       <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
diff --git a/src/itemStore.ts b/src/itemStore.ts
index 05adc3a..388ae1c 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -1,8 +1,9 @@
 import SQLite, {SQLResultSet, SQLTransaction, WebsqlDatabase} from 'react-native-sqlite-2';
 import {Item, ItemAction, itemsReducer} from './items';
+import {mutationHash, mutationTarget, newMutationIdentity} from './mutationProtocol';
 
 export const DATABASE_NAME = 'items.db';
-export const SCHEMA_VERSION = 3;
+export const SCHEMA_VERSION = 4;
 
 export type ItemRow = {
   id: string;
@@ -37,14 +38,22 @@ type UploadOperation =
   | {kind: 'toggle'; itemId: string; payload: {completed: boolean}}
   | {kind: 'delete'; itemId: string; payload: null};
 
-export type PendingMutation = UploadOperation & {sequence: number};
+export type PendingMutation = UploadOperation & {
+  sequence: number;
+  clientMutationId: string;
+  payloadHash: string;
+  terminalError: 'identity_conflict' | null;
+};
+
+export type MutationResult = {item: Item} | {deletedId: string};
 
 export interface ItemStore {
   read(): Promise<Item[]>;
   readLastSuccessfulRefresh(): Promise<number | null>;
   readPending(): Promise<PendingMutation[]>;
   mutate(action: ItemMutation, identityPrefix?: string): Promise<Item[]>;
-  acknowledge(sequence: number): Promise<void>;
+  acknowledge(operation: PendingMutation, result: MutationResult): Promise<void>;
+  rejectIdentity(operation: PendingMutation): Promise<void>;
   replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void>;
 }
 
@@ -58,22 +67,41 @@ function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
   });
 }
 
+function decodeOperation(row: {sequence: number; kind: string; item_id: string; payload: string | null}) {
+  const payload = row.payload === null ? null : JSON.parse(row.payload);
+  const title = payload && typeof payload.title === 'string' && payload.title.trim();
+  const validPayload = row.kind === 'delete' ? payload === null
+    : row.kind === 'create' ? title && payload.id === row.item_id && typeof payload.completed === 'boolean' && Object.keys(payload).length === 3
+      : row.kind === 'rename' ? title && Object.keys(payload).length === 1
+        : row.kind === 'toggle' && payload && typeof payload.completed === 'boolean' && Object.keys(payload).length === 1;
+  if (!Number.isSafeInteger(row.sequence) || row.sequence < 1
+      || typeof row.item_id !== 'string' || !row.item_id || !validPayload) {
+    throw new Error('Invalid pending mutation in the local database');
+  }
+  return {sequence: row.sequence, kind: row.kind, itemId: row.item_id, payload} as UploadOperation & {sequence: number};
+}
+
+function identityFor(operation: UploadOperation, makeIdentity: () => string) {
+  const clientMutationId = makeIdentity();
+  if (typeof clientMutationId !== 'string' || !clientMutationId) {throw new Error('Invalid mutation identity');}
+  const target = mutationTarget(operation.kind, operation.itemId);
+  return {clientMutationId, payloadHash: mutationHash(target.method, target.path, operation.payload)};
+}
+
 function readPending(tx: SQLTransaction, callback: (pending: PendingMutation[]) => void) {
-  tx.executeSql('SELECT sequence, kind, item_id, payload FROM pending_mutations ORDER BY sequence', [], (_, result) => {
+  tx.executeSql('SELECT sequence, kind, item_id, payload, client_mutation_id, payload_hash, terminal_error FROM pending_mutations ORDER BY sequence', [], (_, result) => {
     const pending: PendingMutation[] = [];
     for (let i = 0; i < result.rows.length; i++) {
       const row = result.rows.item(i);
-      const payload = row.payload === null ? null : JSON.parse(row.payload);
-      const title = payload && typeof payload.title === 'string' && payload.title.trim();
-      const validPayload = row.kind === 'delete' ? payload === null
-        : row.kind === 'create' ? title && payload.id === row.item_id && typeof payload.completed === 'boolean'
-          : row.kind === 'rename' ? title && Object.keys(payload).length === 1
-            : row.kind === 'toggle' && payload && typeof payload.completed === 'boolean' && Object.keys(payload).length === 1;
-      if (!Number.isSafeInteger(row.sequence) || row.sequence < 1
-          || typeof row.item_id !== 'string' || !row.item_id || !validPayload) {
-        throw new Error('Invalid pending mutation in the local database');
+      const operation = decodeOperation(row);
+      const target = mutationTarget(operation.kind, operation.itemId);
+      if (typeof row.client_mutation_id !== 'string' || !row.client_mutation_id
+          || row.payload_hash !== mutationHash(target.method, target.path, operation.payload)
+          || (row.terminal_error !== null && row.terminal_error !== 'identity_conflict')) {
+        throw new Error('Invalid durable mutation identity or payload hash');
       }
-      pending.push({sequence: row.sequence, kind: row.kind, itemId: row.item_id, payload} as PendingMutation);
+      pending.push({...operation, clientMutationId: row.client_mutation_id,
+        payloadHash: row.payload_hash, terminalError: row.terminal_error});
     }
     callback(pending);
   });
@@ -82,7 +110,7 @@ function readPending(tx: SQLTransaction, callback: (pending: PendingMutation[])
 // Resolve only in the transaction's success callback, after native COMMIT.
 // A failed statement/commit rolls back and must not publish its candidate state.
 class SqliteItemStore implements ItemStore {
-  constructor(private readonly database: WebsqlDatabase) {}
+  constructor(private readonly database: WebsqlDatabase, private readonly makeIdentity: () => string) {}
 
   read(): Promise<Item[]> {
     return new Promise((resolve, reject) => {
@@ -113,10 +141,38 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
-  acknowledge(sequence: number): Promise<void> {
+  acknowledge(operation: PendingMutation, result: MutationResult): Promise<void> {
+    const resultId = 'item' in result ? result.item.id : result.deletedId;
+    if (resultId !== operation.itemId || (operation.kind === 'delete') !== ('deletedId' in result)) {
+      return Promise.reject(new Error('Acknowledged result does not match pending mutation'));
+    }
+    return new Promise((resolve, reject) => {
+      this.database.transaction(tx => {
+        readPending(tx, pending => {
+          const first = pending[0];
+          if (!first || first.sequence !== operation.sequence || first.clientMutationId !== operation.clientMutationId
+              || first.payloadHash !== operation.payloadHash || first.terminalError !== null) {
+            throw new Error('Acknowledgment does not match the next pending mutation');
+          }
+          // Record the actual result before dequeuing in the SAME native commit.
+          // Do not overwrite later optimistic edits with an earlier response.
+          const acknowledgement = {clientMutationId: first.clientMutationId, payloadHash: first.payloadHash,
+            status: mutationTarget(first.kind, first.itemId).status, result};
+          tx.executeSql('UPDATE sync_metadata SET last_acknowledgement = ? WHERE singleton = 1', [JSON.stringify(acknowledgement)]);
+          tx.executeSql('DELETE FROM pending_mutations WHERE sequence = ? AND client_mutation_id = ? AND payload_hash = ?',
+            [first.sequence, first.clientMutationId, first.payloadHash]);
+        });
+      }, reject, resolve);
+    });
+  }
+
+  rejectIdentity(operation: PendingMutation): Promise<void> {
     return new Promise((resolve, reject) => {
       this.database.transaction(tx => {
-        tx.executeSql('DELETE FROM pending_mutations WHERE sequence = ?', [sequence]);
+        tx.executeSql("UPDATE pending_mutations SET terminal_error = 'identity_conflict' WHERE sequence = ? AND client_mutation_id = ? AND payload_hash = ? AND terminal_error IS NULL",
+          [operation.sequence, operation.clientMutationId, operation.payloadHash], (_, result) => {
+            if (result.rowsAffected !== 1) {throw new Error('Identity conflict does not match pending mutation');}
+          });
       }, reject, resolve);
     });
   }
@@ -179,8 +235,10 @@ class SqliteItemStore implements ItemStore {
             if (operation) {
               // Store every successful user operation in the same transaction
               // as its Item change (and identity allocation). Never coalesce.
-              tx.executeSql('INSERT INTO pending_mutations (kind, item_id, payload) VALUES (?, ?, ?)',
-                [operation.kind, operation.itemId, operation.payload === null ? null : JSON.stringify(operation.payload)]);
+              const identity = identityFor(operation, this.makeIdentity);
+              tx.executeSql('INSERT INTO pending_mutations (kind, item_id, payload, client_mutation_id, payload_hash) VALUES (?, ?, ?, ?, ?)',
+                [operation.kind, operation.itemId, operation.payload === null ? null : JSON.stringify(operation.payload),
+                  identity.clientMutationId, identity.payloadHash]);
             }
             readItems(tx, result => {committed = result;});
           };
@@ -210,13 +268,13 @@ class SqliteItemStore implements ItemStore {
   }
 }
 
-export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
+export async function openItemStore(name = DATABASE_NAME, makeIdentity = newMutationIdentity): Promise<ItemStore> {
   const database = SQLite.openDatabase(name);
   await new Promise<void>((resolve, reject) => {
     database.transaction(tx => {
       tx.executeSql('PRAGMA user_version', [], (_, result) => {
         const version = result.rows.item(0).user_version;
-        if (![0, 1, 2, SCHEMA_VERSION].includes(version)) {
+        if (![0, 1, 2, 3, SCHEMA_VERSION].includes(version)) {
           throw new Error(`Unsupported local database schema ${version}`);
         }
         if (version === 0) {
@@ -234,10 +292,27 @@ export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
           // Existing cached Items, refresh time and local identity are preserved.
           // The sequence orders local intent; it is not remote mutation identity.
           tx.executeSql("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL CHECK(kind IN ('create', 'rename', 'toggle', 'delete')), item_id TEXT NOT NULL, payload TEXT CHECK((kind = 'delete' AND payload IS NULL) OR (kind != 'delete' AND payload IS NOT NULL)))");
-          tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
+        }
+        if (version < 4) {
+          // Fill identities for legacy M05 intent before committing the upgrade.
+          // ALTER preserves the queue's existing AUTOINCREMENT high-water mark.
+          tx.executeSql("ALTER TABLE pending_mutations ADD COLUMN client_mutation_id TEXT NOT NULL DEFAULT ''");
+          tx.executeSql("ALTER TABLE pending_mutations ADD COLUMN payload_hash TEXT NOT NULL DEFAULT ''");
+          tx.executeSql("ALTER TABLE pending_mutations ADD COLUMN terminal_error TEXT CHECK(terminal_error IS NULL OR terminal_error = 'identity_conflict')");
+          tx.executeSql('ALTER TABLE sync_metadata ADD COLUMN last_acknowledgement TEXT');
+          tx.executeSql('SELECT sequence, kind, item_id, payload FROM pending_mutations ORDER BY sequence', [], (_, legacy) => {
+            for (let i = 0; i < legacy.rows.length; i++) {
+              const operation = decodeOperation(legacy.rows.item(i));
+              const identity = identityFor(operation, makeIdentity);
+              tx.executeSql('UPDATE pending_mutations SET client_mutation_id = ?, payload_hash = ? WHERE sequence = ?',
+                [identity.clientMutationId, identity.payloadHash, operation.sequence]);
+            }
+            tx.executeSql('CREATE UNIQUE INDEX pending_mutation_identity ON pending_mutations (client_mutation_id)');
+            tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
+          });
         }
       });
     }, reject, resolve);
   });
-  return new SqliteItemStore(database);
+  return new SqliteItemStore(database, makeIdentity);
 }
diff --git a/src/mutationProtocol.ts b/src/mutationProtocol.ts
new file mode 100644
index 0000000..19da056
--- /dev/null
+++ b/src/mutationProtocol.ts
@@ -0,0 +1,34 @@
+import {sha256} from 'js-sha256';
+
+// Build text directly: inserting sorted numeric keys into an object and then
+// JSON.stringify-ing it would reorder those keys numerically rather than lexically.
+export function canonicalJson(value: unknown): string {
+  if (value === null || typeof value === 'string' || typeof value === 'boolean') {
+    return JSON.stringify(value);
+  }
+  if (typeof value === 'number' && Number.isSafeInteger(value)) {return JSON.stringify(value);}
+  if (Array.isArray(value)) {return `[${value.map(canonicalJson).join(',')}]`;}
+  if (typeof value === 'object' && value !== null
+      && (Object.getPrototypeOf(value) === Object.prototype || Object.getPrototypeOf(value) === null)) {
+    const object = value as Record<string, unknown>;
+    return `{${Object.keys(object).sort().map(key => `${JSON.stringify(key)}:${canonicalJson(object[key])}`).join(',')}}`;
+  }
+  throw new Error('Mutation payload must contain JSON values and safe integers');
+}
+
+export function mutationHash(method: string, path: string, payload: unknown): string {
+  // The instance API exercises the same UTF-8 JavaScript code on host and Hermes;
+  // it does not substitute Node's crypto fast path during host verification.
+  return sha256.create().update(canonicalJson({method, path, payload})).hex();
+}
+
+export function mutationTarget(kind: 'create' | 'rename' | 'toggle' | 'delete', itemId: string) {
+  return kind === 'create' ? {method: 'POST', path: '/items', status: 201}
+    : {method: kind === 'delete' ? 'DELETE' : 'PATCH', path: `/items/${encodeURIComponent(itemId)}`, status: 200};
+}
+
+export function newMutationIdentity(): string {
+  // A new nonce per user operation; its full value is committed before any send.
+  // This is an identity namespace, not a secret or security credential.
+  return `mutation-${Date.now().toString(36)}-${Math.random().toString(36).slice(2)}-${Math.random().toString(36).slice(2)}`;
+}
diff --git a/src/sync.ts b/src/sync.ts
index 9e0bff8..fde8e37 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -1,5 +1,6 @@
 import {Item} from './items';
-import {ItemStore} from './itemStore';
+import {ItemStore, MutationResult, PendingMutation} from './itemStore';
+import {mutationTarget} from './mutationProtocol';
 
 export const FIXTURE_URL = 'http://10.0.2.2:18081';
 
@@ -43,12 +44,19 @@ export class ForegroundSync implements SyncSession {
 
   get initialized() {return this.refreshed;}
 
-  private async exchange(method: string, path: string, expectedStatus: number, body?: unknown) {
+  private async exchange(method: string, path: string, expectedStatus: number, body?: unknown, operation?: PendingMutation) {
     const response = await this.request(`${this.baseUrl}${path}`, {
       method, headers: {'Content-Type': 'application/json'},
       ...(body === undefined ? {} : {body: JSON.stringify(body)}),
     });
     if (response.status !== expectedStatus) {
+      if (operation && response.status === 409) {
+        const failure = await response.json() as {error?: string};
+        if (failure?.error === 'identity_conflict') {
+          await this.store.rejectIdentity(operation);
+          throw new Error('Mutation identity conflict; upload stopped without retry');
+        }
+      }
       throw new Error(`${method} ${path} failed (HTTP ${response.status})`);
     }
     return response.json();
@@ -59,16 +67,22 @@ export class ForegroundSync implements SyncSession {
     // after process restart. Each edit is sent separately, without coalescing.
     const pending = await this.store.readPending();
     for (const operation of pending) {
-      if (operation.kind === 'create') {
-        await this.exchange('POST', '/items', 201, operation.payload);
+      if (operation.terminalError === 'identity_conflict') {
+        throw new Error('Mutation identity conflict; upload stopped without retry');
+      }
+      const target = mutationTarget(operation.kind, operation.itemId);
+      const response = await this.exchange(target.method, target.path, target.status,
+        {...operation.payload, clientMutationId: operation.clientMutationId, payloadHash: operation.payloadHash}, operation) as {item?: unknown; deletedId?: unknown};
+      let result: MutationResult;
+      if (operation.kind === 'delete') {
+        if (response?.deletedId !== operation.itemId) {throw new Error('Invalid remote deletion acknowledgment');}
+        result = {deletedId: operation.itemId};
       } else {
-        const path = `/items/${encodeURIComponent(operation.itemId)}`;
-        await this.exchange(operation.kind === 'delete' ? 'DELETE' : 'PATCH', path, 200,
-          operation.payload === null ? undefined : operation.payload);
+        const item = remoteItem(response?.item);
+        if (item.id !== operation.itemId) {throw new Error('Remote acknowledgment belongs to another Item');}
+        result = {item};
       }
-      // M05 removes successful foreground requests. A process crash between
-      // remote acceptance and this commit still requires M06 idempotency.
-      await this.store.acknowledge(operation.sequence);
+      await this.store.acknowledge(operation, result);
     }
 
     const response = await this.exchange('GET', '/items', 200) as {items?: unknown};
diff --git a/verification/M06.md b/verification/M06.md
index 8fd23b5..f4f617c 100644
--- a/verification/M06.md
+++ b/verification/M06.md
@@ -69,5 +69,55 @@ under the raw root. The earlier malformed read failure is retained at
 | `harness-syntax-01` | Python AST syntax check, exit0 |
 | `baseline-android-01` | Expected M05 lost-ack limitation and cleanup verified, exit0 |
 
-Implementation, host checks and main's final fixed Android run remain pending at
-this reproduction commit. No M07 or phase-2 feature is included.
+Implementation, host checks and main's final fixed Android run were pending at
+the reproduction commit. No M07 or phase-2 feature is included.
+
+## Implemented candidate and host verification
+
+Schema4 commits mutation identity/hash with each Item edit and intent. Migration
+preserves old intent/order and fills identity/hash atomically. A successful
+response records its identity/hash/status/result in `sync_metadata` and removes
+the matching head intent in the same transaction. Later optimistic edits are
+not overwritten by earlier acknowledgments; the final GET supplies canonical
+Items. Identity409 is persisted as a terminal marker and never resent. The debug
+M06 identifier override exercises these production transactions and RN fetch.
+Only the pinned pure-JavaScript `js-sha256`0.11.1 dependency was added; its instance
+API uses the same UTF-8 code on host and Hermes. The fixture hashes independently.
+
+All supporting tests were captured before execution in
+`supporting-tests-before-first-run.json`. Prior scenario values/assertions remain;
+current schema expectations advance3→4, unsupported-version coverage4→5, and
+identity/hash fields augment exact prior payload expectations.
+
+| Invocation | Result |
+|---|---|
+| `dependency-install-01` | Pinned dependency installed with scripts disabled, exit0 |
+| `typecheck-01`, `harness-syntax-02` | TypeScript and Python harness syntax pass, exit0 |
+| `jest-01` | 54/56 pass; two prior DELETE regressions return HTTP400 |
+| `delete-framing-diagnostic-01` | Confirms missing Node HTTP body framing, exit0 |
+| `jest-02` | Full suite56/56 pass, including both affected regressions, exit0 |
+| `typecheck-02` | Final adapter/source typecheck pass, exit0 |
+| `build-01` | Both debug APKs build;99 tasks,12 executed;31.163seconds, exit0 |
+
+The initial host adapter sent DELETE bytes without Content-Length or transfer
+encoding. Raw headers prove Node parsed them as a malformed next request; the
+fixture saw a legacy empty-body delete, while the client received parser400.
+Main approved the adapter-only correction to declare the UTF-8 byte length.
+No fixture, frozen Android harness/input, assertion or RN production framing was
+changed. The exact diff, before/after hashes and diagnostic wire evidence are in
+[host-adapter-correction.json](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/host-adapter-correction.json)
+and adjacent `.diff`/`delete-framing-diagnostic-01.log`; the54/56 failure is retained.
+
+The56 passing tests include all prior scenarios, five independently frozen hash
+vectors, exact original duplicate response, durable terminal collision, actual
+body hash rejection, invalid acknowledgment retention, three acknowledgment
+rollback points, three migration rollback points and the existing twelve atomic
+Item/intent failure cases extended to identity/hash. Host transactions use the
+existing real SQLite file bridge; HTTP integration uses the loopback fixture.
+Full structured results: [jest-02-results.json](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M06/jest-02-results.json).
+
+The executed M06 fixture/harness/inputs remain byte-identical to the baseline
+freeze. Candidate hashes and copied APKs will be in `candidate-manifest.json`.
+Main owns final fixed Android verification on that frozen candidate; it has not
+yet run. This owner will not duplicate it and will change only this ledger after
+main returns the actual result.


