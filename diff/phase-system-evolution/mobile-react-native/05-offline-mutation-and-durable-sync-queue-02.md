## `feat: commit local edits with ordered durable upload intent`

diff --git a/TRACK.md b/TRACK.md
index 9f08875..83dea7e 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -79,6 +79,36 @@ explicit remote-title controls. No native module or new dependency is needed.
 Persisting the refresh time does not preserve upload intent: M03's session-only
 edit comparison and first-pull replacement limits remain unchanged.
 
+## M05: offline mutations and durable foreground upload
+
+Schema v3 adds an ordered `pending_mutations` table. Each changed Item and its
+exact create/rename/completion/delete payload commit in one native SQLite
+transaction, including identity allocation. Failed statements or commit roll
+back both halves; no-op edits add no intent. Schema-v1/v2 migration preserves
+existing Items, allocator and refresh metadata; old unsent intent cannot be
+reconstructed from M04's snapshot alone.
+
+Foreground sync now reads this queue from storage, sends every operation in
+sequence, removes each successfully acknowledged entry and then commits a remote
+snapshot. It never coalesces rename and completion. A pending queue prevents a
+snapshot replacement from erasing unsent local edits. App startup still does no
+network work, and both list and pending count recover from SQLite after actual
+offline process termination. Fresh/stale/error semantics from M04 remain intact.
+
+M05 supersedes M03's first-sync replacement of unsent edits. Every new UI Item can
+now upload, so creation uses the already-existing distinct session namespace even
+before the first refresh. IDs persist in full; the deterministic Android `device`
+prefix remains debug-only. M01/M02 host tests inject their same original `item`
+prefix explicitly; the CRUD sequence, values and stable identities are unchanged.
+
+This remains an explicit foreground mechanism. The local sequence is not a remote
+mutation identity. A crash between remote acceptance and local dequeue can still
+repeat a side effect; M06 owns that guarantee. No idempotency, conflict handling,
+automatic retry/backoff, scheduler, push, state library, dependency or native
+module is added. `scripts/verify_m05.py` freezes the actual offline four-edit,
+process-kill/relaunch and one-drain scenario; see `verification/M05.md` for hashes,
+raw failures and the coordinator's final Android verification.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
@@ -102,6 +132,8 @@ python3 scripts/verify_m02.py --apk android/app/build/outputs/apk/debug/app-debu
 python3 scripts/verify_m03.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m03
 # M04 uses the same fixed emulator and owns/restores its fixture/connectivity.
 python3 scripts/verify_m04.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m04
+# M05 uses the exact same emulator/fixture; one ordered drain follows offline death.
+python3 scripts/verify_m05.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m05
 # To run the fixture separately for manual foreground sync:
 node fixture/server.cjs
 ```
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 509c97b..04d3f02 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -11,7 +11,7 @@ test('M01 fixed sequence maps stable Item identity to the rendered list', async
   let clock = 1700000000000;
   const clockSpy = jest.spyOn(Date, 'now').mockImplementation(() => {const value = clock; clock += 1000; return value;});
   try {
-    render(<App />);
+    render(<App testIdentityPrefix="item" />);
     await saved();
     expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
     fireEvent.changeText(screen.getByLabelText('New item title'), 'Alpha');
@@ -53,7 +53,7 @@ test('M01 fixed sequence maps stable Item identity to the rendered list', async
 test('M02 startup reads saved Items; failed commits leave the draft and confirmed UI unchanged', async () => {
   const store = await openItemStore();
   const original = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  render(<App />);
+  render(<App testIdentityPrefix="item" />);
   await saved();
   expect(screen.getByText('Alpha')).toBeTruthy();
   failNextSql(/^END/);
@@ -235,3 +235,85 @@ test('M04 a committed local edit becomes stale without changing last successful
   expect(screen.getByLabelText('Last successful refresh: 1700000200000')).toBeTruthy();
   expect(await store.read()).toEqual([{...m04Seeds[0], title: 'Locally revised', version: 2, updatedAt: 1700000202000}, m04Seeds[1]]);
 });
+
+test('M05 UI reads durable pending work after restart and clears it after a foreground drain', async () => {
+  const gamma = {id: 'device-001', title: 'Gamma', completed: false, version: 1, updatedAt: 1700000300000};
+  const renamed = {...m04Seeds[0], title: 'Queued edit', version: 2, updatedAt: 1700000301000};
+  const completed = {...renamed, completed: true, version: 3, updatedAt: 1700000302000};
+  const final = [gamma, completed];
+  // Component test transport only; the separate host HTTP and Android harness
+  // execute the real fixture. Native-bridge host SQLite transactions remain real.
+  const replies = [
+    {method: 'POST', status: 201, body: {id: 'device-001', title: 'Gamma', completed: false}, response: {item: gamma}},
+    {method: 'PATCH', status: 200, body: {title: 'Queued edit'}, response: {item: renamed}},
+    {method: 'PATCH', status: 200, body: {completed: true}, response: {item: completed}},
+    {method: 'DELETE', status: 200, body: null, response: {deletedId: 'remote-002'}},
+    {method: 'GET', status: 200, body: null, response: {items: final}},
+  ];
+  let offline = true;
+  const request: JsonRequest = jest.fn(async (_url, options) => {
+    if (offline) {throw new TypeError('Network request failed');}
+    const reply = replies.shift();
+    expect(reply).toBeDefined();
+    expect(options.method).toBe(reply!.method);
+    expect(options.body ? JSON.parse(options.body) : null).toEqual(reply!.body);
+    return {status: reply!.status, json: async () => reply!.response};
+  });
+  const store = await openItemStore();
+  await store.replaceSnapshot(m04Seeds, 1700000200000);
+  const createSync = (opened: typeof store) => new ForegroundSync(opened, 'http://fixed-m05', request, 'device', () => 1700000304000);
+  const view = render(<App openStore={async () => store} createSync={createSync} />);
+  await saved();
+  expect(screen.getByLabelText('Pending uploads: 0')).toBeTruthy();
+  const clock = jest.spyOn(Date, 'now').mockReturnValue(1700000300000);
+  try {
+    failNextSql(/^INSERT INTO pending_mutations/);
+    fireEvent.changeText(screen.getByLabelText('New item title'), 'Gamma');
+    fireEvent.press(screen.getByLabelText('Add item'));
+    await waitFor(() => expect(screen.getByLabelText('Local storage error')).toBeTruthy());
+    expect(screen.getByText('Change not saved')).toBeTruthy();
+    expect(screen.getByLabelText('New item title').props.value).toBe('Gamma');
+    expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
+    expect(screen.getByLabelText('Pending uploads: 0')).toBeTruthy();
+    expect(await store.read()).toEqual(m04Seeds);
+    fireEvent.press(screen.getByLabelText('Add item'));
+    await saved();
+    expect(screen.getByLabelText('Pending uploads: 1')).toBeTruthy();
+    clock.mockReturnValue(1700000301000);
+    fireEvent.press(screen.getByLabelText('Edit Alpha'));
+    fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Queued edit');
+    fireEvent.press(screen.getByLabelText('Save title'));
+    await saved();
+    expect(screen.getByLabelText('Pending uploads: 2')).toBeTruthy();
+    clock.mockReturnValue(1700000302000);
+    fireEvent.press(screen.getByLabelText('Mark Queued edit complete'));
+    await saved();
+    expect(screen.getByLabelText('Pending uploads: 3')).toBeTruthy();
+    fireEvent.press(screen.getByLabelText('Delete Beta'));
+    await saved();
+    expect(screen.getByLabelText('Pending uploads: 4')).toBeTruthy();
+    expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
+    expect(screen.queryByText('Beta')).toBeNull();
+    expect(request).not.toHaveBeenCalled();
+    view.unmount();
+    closeDatabases();
+    const reopened = await openItemStore();
+    render(<App openStore={async () => reopened} createSync={createSync} />);
+    await saved();
+    expect(screen.getByLabelText('Sync status: stale')).toBeTruthy();
+    expect(screen.getByLabelText('Pending uploads: 4')).toBeTruthy();
+    expect(screen.getByText('Gamma')).toBeTruthy();
+    expect(screen.getByRole('checkbox', {name: 'Mark Queued edit incomplete'}).props.accessibilityState.checked).toBe(true);
+    expect(screen.queryByText('Beta')).toBeNull();
+    expect(request).not.toHaveBeenCalled();
+    offline = false;
+    fireEvent.press(screen.getByLabelText('Synchronize'));
+    await saved();
+    expect(screen.getByLabelText('Sync status: fresh')).toBeTruthy();
+    expect(screen.getByLabelText('Pending uploads: 0')).toBeTruthy();
+    expect(await reopened.read()).toEqual(final);
+    expect(await reopened.readPending()).toEqual([]);
+    expect(replies).toEqual([]);
+    expect(request).toHaveBeenCalledTimes(5);
+  } finally {clock.mockRestore();}
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index bc923c1..384440a 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -1,6 +1,6 @@
 import {Item, itemsReducer} from '../src/items';
 import SQLite from 'react-native-sqlite-2';
-import {itemToRow, openItemStore, rowToItem} from '../src/itemStore';
+import {ItemMutation, itemToRow, openItemStore, PendingMutation, rowToItem} from '../src/itemStore';
 import {closeDatabases, connection, failNextSql} from './sqliteNative';
 
 test('M01 fixed sequence preserves first identity and all Item fields', () => {
@@ -82,11 +82,11 @@ test.each([/^INSERT INTO items/, /^END/])('M02 failure at %s rolls back Item and
 test('M02 unsupported schema is rejected without recreating or deleting existing data', async () => {
   const store = await openItemStore();
   const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  connection().exec('PRAGMA user_version = 3');
+  connection().exec('PRAGMA user_version = 4');
   closeDatabases();
-  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 3');
+  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 4');
   expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(3);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(4);
 });
 
 test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
@@ -125,7 +125,7 @@ test('M04 upgrades a literal M03 schema without touching cached Items or local i
   const store = await openItemStore(name);
   expect(await store.read()).toEqual(seeds);
   expect(await store.readLastSuccessfulRefresh()).toBeNull();
-  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(2);
+  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(3);
   expect(database.prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
   closeDatabases();
   const reopened = await openItemStore(name);
@@ -133,3 +133,108 @@ test('M04 upgrades a literal M03 schema without touching cached Items or local i
   expect(await reopened.readLastSuccessfulRefresh()).toBeNull();
   expect(connection(name).prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
 });
+
+const m05Seeds: Item[] = [
+  {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
+  {id: 'remote-002', title: 'Beta', completed: false, version: 1, updatedAt: 1700000000000},
+];
+const m05Actions: ItemMutation[] = [
+  {type: 'create', title: 'Gamma', now: 1700000300000},
+  {type: 'rename', id: 'remote-001', title: 'Queued edit', now: 1700000301000},
+  {type: 'toggle', id: 'remote-001', now: 1700000302000},
+  {type: 'delete', id: 'remote-002'},
+];
+const m05Pending: PendingMutation[] = [
+  {sequence: 1, kind: 'create', itemId: 'device-001', payload: {id: 'device-001', title: 'Gamma', completed: false}},
+  {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
+  {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
+  {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
+];
+
+const mutationFailures = [
+  {index: 0, kind: 'create', localSql: /^INSERT INTO items/},
+  {index: 1, kind: 'rename', localSql: /^UPDATE items/},
+  {index: 2, kind: 'toggle', localSql: /^UPDATE items/},
+  {index: 3, kind: 'delete', localSql: /^DELETE FROM items WHERE/},
+].flatMap(operation => [operation.localSql, /^INSERT INTO pending_mutations/, /^END/]
+  .map(failAt => ({...operation, failAt})));
+
+test.each(mutationFailures)('M05 $kind rollback at $failAt preserves both Item and ordered intent', async ({index, failAt}) => {
+  const store = await openItemStore();
+  await store.replaceSnapshot(m05Seeds, 1700000200000);
+  for (const action of m05Actions.slice(0, index)) {await store.mutate(action, 'device');}
+  const before = await store.read();
+  const beforePending = await store.readPending();
+  const identity = connection().prepare('SELECT next_id FROM local_identity WHERE singleton=1').get()?.next_id;
+  expect(beforePending).toEqual(m05Pending.slice(0, index));
+  failNextSql(failAt);
+  await expect(store.mutate(m05Actions[index], 'device')).rejects.toThrow('injected persistence failure');
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(before);
+  expect(await reopened.readPending()).toEqual(beforePending);
+  expect(connection().prepare('SELECT next_id FROM local_identity WHERE singleton=1').get()?.next_id).toBe(identity);
+  await reopened.mutate(m05Actions[index], 'device');
+  expect(await reopened.readPending()).toEqual(m05Pending.slice(0, index + 1));
+  console.info('M05_ATOMIC_ROLLBACK', JSON.stringify({operation: m05Actions[index], failure: String(failAt),
+    before, beforePending, afterRetry: await reopened.read(), pendingAfterRetry: await reopened.readPending()}));
+});
+
+test('M05 no-op mutations do not create false upload intent', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot(m05Seeds);
+  await store.mutate({type: 'create', title: ' ', now: 1700000300000});
+  await store.mutate({type: 'rename', id: 'remote-001', title: ' ', now: 1700000301000});
+  await store.mutate({type: 'rename', id: 'remote-001', title: 'Alpha', now: 1700000301000});
+  await store.mutate({type: 'toggle', id: 'missing', now: 1700000302000});
+  await store.mutate({type: 'delete', id: 'missing'});
+  expect(await store.read()).toEqual(m05Seeds);
+  expect(await store.readPending()).toEqual([]);
+  expect(connection().prepare('SELECT next_id FROM local_identity WHERE singleton=1').get()?.next_id).toBe(1);
+});
+
+test('M05 a snapshot cannot replace unuploaded local edits or clear their intent', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot(m05Seeds, 1700000200000);
+  for (const action of m05Actions) {await store.mutate(action, 'device');}
+  const local = await store.read();
+  await expect(store.replaceSnapshot(m05Seeds, 1700000202000)).rejects.toThrow('Pending uploads must drain');
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(local);
+  expect(await reopened.readPending()).toEqual(m05Pending);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
+});
+
+test('M05 v2 migration preserves Items, allocator and refresh time, including failed schema creation', async () => {
+  const database = connection();
+  database.exec(`
+    CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL);
+    CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0));
+    INSERT INTO local_identity (singleton, next_id) VALUES (1, 3);
+    CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), last_successful_refresh_at INTEGER CHECK(last_successful_refresh_at IS NULL OR last_successful_refresh_at >= 0));
+    INSERT INTO sync_metadata (singleton, last_successful_refresh_at) VALUES (1, 1700000200000);
+    PRAGMA user_version = 2;
+  `);
+  for (const item of m05Seeds) {
+    database.prepare('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)')
+      .run(item.id, item.title, Number(item.completed), item.version, item.updatedAt);
+  }
+  failNextSql(/^CREATE TABLE pending_mutations/);
+  await expect(openItemStore()).rejects.toThrow('injected persistence failure');
+  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(2);
+  expect(database.prepare('SELECT * FROM items ORDER BY id').all()).toEqual(m05Seeds.map(itemToRow));
+  expect(database.prepare("SELECT name FROM sqlite_master WHERE name='pending_mutations'").get()).toBeUndefined();
+  closeDatabases();
+  const migrated = await openItemStore();
+  expect(await migrated.read()).toEqual(m05Seeds);
+  expect(await migrated.readPending()).toEqual([]);
+  expect(await migrated.readLastSuccessfulRefresh()).toBe(1700000200000);
+  expect(connection().prepare('SELECT next_id FROM local_identity WHERE singleton=1').get()?.next_id).toBe(3);
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(m05Seeds);
+  expect(await reopened.readPending()).toEqual([]);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(3);
+});
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index baf2d66..0c0dc28 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -236,3 +236,33 @@ test('M05 fixed four offline mutations survive reopening and drain as four accep
     {method: 'GET', path: '/items', body: null, status: 200, response: {items: expected}},
   ]});
 });
+
+test('M05 one failed offline foreground drain retains every ordered intent for reopening', async () => {
+  await control('/__mutation-clock', {nextTimestamp: 1700000300000});
+  let offline = false;
+  const transport: JsonRequest = (address, options) => offline
+    ? Promise.reject(new TypeError('Network request failed')) : request(address, options);
+  const store = await openItemStore();
+  const sync = new ForegroundSync(store, url, transport, 'device');
+  await sync.synchronize();
+  offline = true;
+  await store.mutate({type: 'create', title: 'Gamma', now: 1700000300000}, 'device');
+  await store.mutate({type: 'rename', id: 'remote-001', title: 'Queued edit', now: 1700000301000});
+  await store.mutate({type: 'toggle', id: 'remote-001', now: 1700000302000});
+  await store.mutate({type: 'delete', id: 'remote-002'});
+  const local = await store.read();
+  const pending = await store.readPending();
+  expect(pending).toEqual([
+    {sequence: 1, kind: 'create', itemId: 'device-001', payload: {id: 'device-001', title: 'Gamma', completed: false}},
+    {sequence: 2, kind: 'rename', itemId: 'remote-001', payload: {title: 'Queued edit'}},
+    {sequence: 3, kind: 'toggle', itemId: 'remote-001', payload: {completed: true}},
+    {sequence: 4, kind: 'delete', itemId: 'remote-002', payload: null},
+  ]);
+  await expect(sync.synchronize()).rejects.toThrow('Network request failed');
+  expect(fixture.state().requests).toHaveLength(1);
+  expect(fixture.state().items).toEqual(seeds);
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(local);
+  expect(await reopened.readPending()).toEqual(pending);
+});
diff --git a/scripts/verify_m02.py b/scripts/verify_m02.py
index e516ce8..214d619 100644
--- a/scripts/verify_m02.py
+++ b/scripts/verify_m02.py
@@ -99,11 +99,11 @@ def main():
         assert path.exists(), "Native database file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as database:
             assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert database.execute("PRAGMA user_version").fetchone()[0] == 2
+            assert database.execute("PRAGMA user_version").fetchone()[0] == 3
             assert [column[1] for column in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in database.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid")]
-            state = {"schema_version": 2, "items": items,
+            state = {"schema_version": 3, "items": items,
                      "next_id": database.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(state, indent=2))
         return state
diff --git a/scripts/verify_m03.py b/scripts/verify_m03.py
index 29685c0..9f1b906 100644
--- a/scripts/verify_m03.py
+++ b/scripts/verify_m03.py
@@ -138,11 +138,11 @@ def main():
         assert path.exists(), "Native SQLite file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert db.execute("PRAGMA user_version").fetchone()[0] == 2
+            assert db.execute("PRAGMA user_version").fetchone()[0] == 3
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
-            value = {"items": items, "schema_version": 2,
+            value = {"items": items, "schema_version": 3,
                      "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
         return value
diff --git a/scripts/verify_m04.py b/scripts/verify_m04.py
index 32ac942..80eea27 100644
--- a/scripts/verify_m04.py
+++ b/scripts/verify_m04.py
@@ -116,7 +116,7 @@ def main():
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
             schema = db.execute("PRAGMA user_version").fetchone()[0]
-            assert schema == (1 if args.baseline else 2), schema
+            assert schema == (1 if args.baseline else 3), schema
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
diff --git a/src/App.tsx b/src/App.tsx
index aa8b6a5..9ceb61e 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -26,6 +26,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const [error, setError] = useState<string | null>(null);
   const [refresh, setRefresh] = useState<RefreshState>({status: 'stale'});
   const [lastSuccessfulRefreshAt, setLastSuccessfulRefreshAt] = useState<number | null>(null);
+  const [pendingCount, setPendingCount] = useState<number | null>(null);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
@@ -41,11 +42,13 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     openStore().then(async opened => {
       const saved = await opened.read();
       const lastRefresh = await opened.readLastSuccessfulRefresh();
+      const pending = await opened.readPending();
       if (active) {
         store.current = opened;
         sync.current = createSync(opened, testIdentityPrefix, testRefreshClock);
         setItems(saved);
         setLastSuccessfulRefreshAt(lastRefresh);
+        setPendingCount(pending.length);
         setRefresh({status: 'stale'});
         setReady(true);
       }
@@ -57,19 +60,27 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     return () => {active = false;};
   }, [openStore, openAttempt, createSync, testIdentityPrefix, testRefreshClock]);
 
+  async function reloadPending() {
+    try {setPendingCount((await store.current!.readPending()).length);}
+    catch {setPendingCount(null);} // A failed status read must not claim a committed edit was unsaved.
+  }
+
   async function mutate(action: ItemMutation): Promise<boolean> {
     if (!store.current || busyRef.current) {return false;}
     busyRef.current = true;
     setBusy(true);
     setError(null);
     try {
-      setItems(await store.current.mutate(action, sync.current?.initialized ? sync.current.identityPrefix : undefined));
+      // Every new Item can now upload, even before the first successful refresh.
+      // Use the existing distinct namespace immediately; full IDs persist in SQL.
+      setItems(await store.current.mutate(action, sync.current?.identityPrefix));
       setRefresh({status: 'stale'});
       return true;
     } catch (reason) {
       setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);
       return false;
     } finally {
+      await reloadPending();
       busyRef.current = false;
       setBusy(false);
     }
@@ -90,6 +101,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     } catch (reason) {
       setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});
     } finally {
+      await reloadPending();
       busyRef.current = false;
       setBusy(false);
     }
@@ -127,12 +139,13 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         <Text accessibilityLabel={`Last successful refresh: ${lastSuccessfulRefreshAt ?? 'never'}`}>
           Last successful refresh: {lastSuccessfulRefreshAt === null ? 'never' : new Date(lastSuccessfulRefreshAt).toISOString()}
         </Text>
+        <Text accessibilityLabel={`Pending uploads: ${pendingCount ?? 'unknown'}`}>
+          {pendingCount === null ? 'Pending upload count unavailable' : `Pending uploads: ${pendingCount}`}
+        </Text>
         {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
       </>}
       <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
-      <Text>{sync.current?.initialized
-        ? 'Sync sends this session's local edits, then pulls the remote snapshot.'
-        : 'First sync pulls the remote snapshot, replacing local Items. Sync edits before closing this session.'}</Text>
+      <Text>Sync uploads saved edits in order, then refreshes Items.</Text>
       <TextInput
         accessibilityLabel={editingId === null ? 'New item title' : 'Edit item title'}
         placeholder="Item title"
diff --git a/src/itemStore.ts b/src/itemStore.ts
index a82dc6c..05adc3a 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -2,7 +2,7 @@ import SQLite, {SQLResultSet, SQLTransaction, WebsqlDatabase} from 'react-native
 import {Item, ItemAction, itemsReducer} from './items';
 
 export const DATABASE_NAME = 'items.db';
-export const SCHEMA_VERSION = 2;
+export const SCHEMA_VERSION = 3;
 
 export type ItemRow = {
   id: string;
@@ -31,10 +31,20 @@ export function rowToItem(row: ItemRow): Item {
 export type ItemMutation = Exclude<ItemAction, {type: 'create'}>
   | {type: 'create'; title: string; now: number};
 
+type UploadOperation =
+  | {kind: 'create'; itemId: string; payload: {id: string; title: string; completed: boolean}}
+  | {kind: 'rename'; itemId: string; payload: {title: string}}
+  | {kind: 'toggle'; itemId: string; payload: {completed: boolean}}
+  | {kind: 'delete'; itemId: string; payload: null};
+
+export type PendingMutation = UploadOperation & {sequence: number};
+
 export interface ItemStore {
   read(): Promise<Item[]>;
   readLastSuccessfulRefresh(): Promise<number | null>;
+  readPending(): Promise<PendingMutation[]>;
   mutate(action: ItemMutation, identityPrefix?: string): Promise<Item[]>;
+  acknowledge(sequence: number): Promise<void>;
   replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void>;
 }
 
@@ -48,6 +58,27 @@ function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
   });
 }
 
+function readPending(tx: SQLTransaction, callback: (pending: PendingMutation[]) => void) {
+  tx.executeSql('SELECT sequence, kind, item_id, payload FROM pending_mutations ORDER BY sequence', [], (_, result) => {
+    const pending: PendingMutation[] = [];
+    for (let i = 0; i < result.rows.length; i++) {
+      const row = result.rows.item(i);
+      const payload = row.payload === null ? null : JSON.parse(row.payload);
+      const title = payload && typeof payload.title === 'string' && payload.title.trim();
+      const validPayload = row.kind === 'delete' ? payload === null
+        : row.kind === 'create' ? title && payload.id === row.item_id && typeof payload.completed === 'boolean'
+          : row.kind === 'rename' ? title && Object.keys(payload).length === 1
+            : row.kind === 'toggle' && payload && typeof payload.completed === 'boolean' && Object.keys(payload).length === 1;
+      if (!Number.isSafeInteger(row.sequence) || row.sequence < 1
+          || typeof row.item_id !== 'string' || !row.item_id || !validPayload) {
+        throw new Error('Invalid pending mutation in the local database');
+      }
+      pending.push({sequence: row.sequence, kind: row.kind, itemId: row.item_id, payload} as PendingMutation);
+    }
+    callback(pending);
+  });
+}
+
 // Resolve only in the transaction's success callback, after native COMMIT.
 // A failed statement/commit rolls back and must not publish its candidate state.
 class SqliteItemStore implements ItemStore {
@@ -75,6 +106,21 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
+  readPending(): Promise<PendingMutation[]> {
+    return new Promise((resolve, reject) => {
+      let pending: PendingMutation[] = [];
+      this.database.readTransaction(tx => readPending(tx, result => {pending = result;}), reject, () => resolve(pending));
+    });
+  }
+
+  acknowledge(sequence: number): Promise<void> {
+    return new Promise((resolve, reject) => {
+      this.database.transaction(tx => {
+        tx.executeSql('DELETE FROM pending_mutations WHERE sequence = ?', [sequence]);
+      }, reject, resolve);
+    });
+  }
+
   replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void> {
     if (lastSuccessfulRefreshAt !== undefined
         && (!Number.isSafeInteger(lastSuccessfulRefreshAt) || lastSuccessfulRefreshAt < 0)) {
@@ -82,18 +128,21 @@ class SqliteItemStore implements ItemStore {
     }
     return new Promise((resolve, reject) => {
       this.database.transaction(tx => {
-        tx.executeSql('DELETE FROM items');
-        for (const item of items) {
-          const row = itemToRow(item);
-          tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
-            [row.id, row.title, row.completed, row.version, row.updated_at]);
-        }
-        if (lastSuccessfulRefreshAt !== undefined) {
-          // The recorded success describes this exact committed snapshot. A
-          // failed Item or metadata statement rolls back both in one transaction.
-          tx.executeSql('UPDATE sync_metadata SET last_successful_refresh_at = ? WHERE singleton = 1',
-            [lastSuccessfulRefreshAt]);
-        }
+        tx.executeSql('SELECT COUNT(*) AS count FROM pending_mutations', [], (_, result) => {
+          // A remote pull cannot erase a local edit that has not uploaded yet.
+          if (result.rows.item(0).count !== 0) {throw new Error('Pending uploads must drain before replacing Items');}
+          tx.executeSql('DELETE FROM items');
+          for (const item of items) {
+            const row = itemToRow(item);
+            tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+              [row.id, row.title, row.completed, row.version, row.updated_at]);
+          }
+          if (lastSuccessfulRefreshAt !== undefined) {
+            // The time describes this exact committed snapshot.
+            tx.executeSql('UPDATE sync_metadata SET last_successful_refresh_at = ? WHERE singleton = 1',
+              [lastSuccessfulRefreshAt]);
+          }
+        });
       }, reject, resolve);
     });
   }
@@ -105,21 +154,34 @@ class SqliteItemStore implements ItemStore {
         readItems(tx, current => {
           const apply = (identified: ItemAction) => {
             const next = itemsReducer(current, identified);
-            if (identified.type === 'delete') {
+            const before = current.find(value => value.id === identified.id);
+            const item = next.find(value => value.id === identified.id);
+            let operation: UploadOperation | null = null;
+            if (identified.type === 'delete' && before) {
               tx.executeSql('DELETE FROM items WHERE id = ?', [identified.id]);
-            } else {
-              const item = next.find(value => value.id === identified.id);
-              if (item && item !== current.find(value => value.id === identified.id)) {
+              operation = {kind: 'delete', itemId: identified.id, payload: null};
+            } else if (identified.type !== 'delete') {
+              if (item && item !== before) {
                 const row = itemToRow(item);
                 if (identified.type === 'create') {
                   tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
                     [row.id, row.title, row.completed, row.version, row.updated_at]);
+                  operation = {kind: 'create', itemId: item.id, payload: {id: item.id, title: item.title, completed: item.completed}};
                 } else {
                   tx.executeSql('UPDATE items SET title = ?, completed = ?, version = ?, updated_at = ? WHERE id = ?',
                     [row.title, row.completed, row.version, row.updated_at, row.id]);
+                  operation = identified.type === 'rename'
+                    ? {kind: 'rename', itemId: item.id, payload: {title: item.title}}
+                    : {kind: 'toggle', itemId: item.id, payload: {completed: item.completed}};
                 }
               }
             }
+            if (operation) {
+              // Store every successful user operation in the same transaction
+              // as its Item change (and identity allocation). Never coalesce.
+              tx.executeSql('INSERT INTO pending_mutations (kind, item_id, payload) VALUES (?, ?, ?)',
+                [operation.kind, operation.itemId, operation.payload === null ? null : JSON.stringify(operation.payload)]);
+            }
             readItems(tx, result => {committed = result;});
           };
 
@@ -154,6 +216,9 @@ export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
     database.transaction(tx => {
       tx.executeSql('PRAGMA user_version', [], (_, result) => {
         const version = result.rows.item(0).user_version;
+        if (![0, 1, 2, SCHEMA_VERSION].includes(version)) {
+          throw new Error(`Unsupported local database schema ${version}`);
+        }
         if (version === 0) {
           tx.executeSql('CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL)');
           tx.executeSql('CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0))');
@@ -164,9 +229,12 @@ export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
           // An old database has no recorded successful-refresh timestamp.
           tx.executeSql('CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), last_successful_refresh_at INTEGER CHECK(last_successful_refresh_at IS NULL OR last_successful_refresh_at >= 0))');
           tx.executeSql('INSERT INTO sync_metadata (singleton, last_successful_refresh_at) VALUES (1, NULL)');
+        }
+        if (version < 3) {
+          // Existing cached Items, refresh time and local identity are preserved.
+          // The sequence orders local intent; it is not remote mutation identity.
+          tx.executeSql("CREATE TABLE pending_mutations (sequence INTEGER PRIMARY KEY AUTOINCREMENT, kind TEXT NOT NULL CHECK(kind IN ('create', 'rename', 'toggle', 'delete')), item_id TEXT NOT NULL, payload TEXT CHECK((kind = 'delete' AND payload IS NULL) OR (kind != 'delete' AND payload IS NOT NULL)))");
           tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
-        } else if (version !== SCHEMA_VERSION) {
-          throw new Error(`Unsupported local database schema ${version}`);
         }
       });
     }, reject, resolve);
diff --git a/src/sync.ts b/src/sync.ts
index 857220d..9e0bff8 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -33,7 +33,7 @@ function sessionIdentity() {
 }
 
 export class ForegroundSync implements SyncSession {
-  private baseline: Item[] | null = null;
+  private refreshed = false;
 
   constructor(private readonly store: ItemStore,
     private readonly baseUrl = FIXTURE_URL,
@@ -41,7 +41,7 @@ export class ForegroundSync implements SyncSession {
     readonly identityPrefix = sessionIdentity(),
     private readonly now: () => number = () => Date.now()) {}
 
-  get initialized() {return this.baseline !== null;}
+  get initialized() {return this.refreshed;}
 
   private async exchange(method: string, path: string, expectedStatus: number, body?: unknown) {
     const response = await this.request(`${this.baseUrl}${path}`, {
@@ -55,29 +55,20 @@ export class ForegroundSync implements SyncSession {
   }
 
   async synchronize(): Promise<void> {
-    // M03 remembers only this foreground session's last shared snapshot, not
-    // durable pending intent. The first explicit sync is a remote pull.
-    if (this.baseline !== null) {
-      const local = await this.store.read();
-      const previous = new Map(this.baseline.map(item => [item.id, item]));
-      for (const item of [...local].sort((a, b) => a.id.localeCompare(b.id))) {
-        const before = previous.get(item.id);
-        if (!before) {
-          await this.exchange('POST', '/items', 201, {id: item.id, title: item.title, completed: item.completed});
-        } else {
-          const patch: {title?: string; completed?: boolean} = {};
-          if (item.title !== before.title) {patch.title = item.title;}
-          if (item.completed !== before.completed) {patch.completed = item.completed;}
-          if (Object.keys(patch).length) {
-            await this.exchange('PATCH', `/items/${encodeURIComponent(item.id)}`, 200, patch);
-          }
-        }
-      }
-      for (const item of [...this.baseline].sort((a, b) => a.id.localeCompare(b.id))) {
-        if (!local.some(value => value.id === item.id)) {
-          await this.exchange('DELETE', `/items/${encodeURIComponent(item.id)}`, 200);
-        }
+    // Recover ordered upload intent from SQLite, including on the first sync
+    // after process restart. Each edit is sent separately, without coalescing.
+    const pending = await this.store.readPending();
+    for (const operation of pending) {
+      if (operation.kind === 'create') {
+        await this.exchange('POST', '/items', 201, operation.payload);
+      } else {
+        const path = `/items/${encodeURIComponent(operation.itemId)}`;
+        await this.exchange(operation.kind === 'delete' ? 'DELETE' : 'PATCH', path, 200,
+          operation.payload === null ? undefined : operation.payload);
       }
+      // M05 removes successful foreground requests. A process crash between
+      // remote acceptance and this commit still requires M06 idempotency.
+      await this.store.acknowledge(operation.sequence);
     }
 
     const response = await this.exchange('GET', '/items', 200) as {items?: unknown};
@@ -87,7 +78,6 @@ export class ForegroundSync implements SyncSession {
       throw new Error('Duplicate Item in remote snapshot');
     }
     await this.store.replaceSnapshot(snapshot, this.now());
-    // Future comparisons and UI publication both come from committed SQLite.
-    this.baseline = await this.store.read();
+    this.refreshed = true;
   }
 }
diff --git a/verification/M05.md b/verification/M05.md
index a14e096..4013797 100644
--- a/verification/M05.md
+++ b/verification/M05.md
@@ -90,3 +90,42 @@ under `snapshots/<invocation>/`. All repeated or failed invocations remain prese
 
 This reproduction commit is intentionally red. Implementation and post-fix
 verification remain pending at this point in history.
+
+## Implemented candidate and host verification
+
+Schema3 stores each exact operation in `pending_mutations`, using a local
+autoincrement sequence. Item change, intent insert and identity allocation share
+one transaction. Foreground sync reads the queue from SQLite, sends every entry
+in order, removes accepted entries and then commits canonical Items plus refresh
+time. Snapshot replacement rejects outstanding intent. Startup and pending count
+read local storage without a network request. No new dependency or native module
+is required. Remote idempotency and the remote-accept/dequeue crash window remain
+explicit M06 work; there is no retry, conflict, background or push mechanism.
+
+`supporting-tests-before-first-run.json` records all supporting test hashes before
+their first execution. The original desired-result test remains byte-identical;
+new tests are appended. Existing M02/M03 Android harnesses change only two current
+schema numbers each, and M04 changes only its current schema assertion2→3.
+Existing unsupported-version coverage advances to4, literal-v1 migration to3.
+The first two local App tests use the existing `testIdentityPrefix="item"` so their
+original identity assertions remain exact; production now applies its existing
+distinct namespace before first sync because all new edits can upload.
+
+| Invocation | Outcome | Exit |
+|---|---|---:|
+| patch-context-01 | Patch rejected on an incorrect old variable name in M02 harness; no files changed; corrected from observed source | tool error |
+| typecheck-01 | `npm run typecheck` | 0 |
+| harness-syntax-02 | Compile all four Python harnesses | 0 |
+| jest-01 | All39 tests/3 suites pass; raw structured results in `jest-01-results.json` | 0 |
+| build-01 | Both debug APKs compile;99 tasks,8 executed/91 up-to-date;26.739seconds | 0 |
+
+The39 tests include all12 Item-write/intent-insert/commit failure combinations,
+SQLite reopen/order, no-op handling, migration rollback, pending-snapshot guard,
+the exact four accepted HTTP operations/final state, offline-drain failure and
+UI pending recovery. Host SQLite uses the existing real file bridge replacement;
+host HTTP uses the actual loopback fixture. The App presentation test's transport
+is explicitly mocked and is not Android or remote-integration evidence.
+
+Per coordinated verification, main will directly execute the final fixed Android
+scenario and relevant regression once on the frozen candidate. No implemented
+Android success is claimed yet and this agent will not duplicate that run.


