## `feat: explain cached refresh state and retain successful time`

diff --git a/TRACK.md b/TRACK.md
index b0612d0..9f08875 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -56,6 +56,29 @@ reads/writes remain persistent, but these upload guarantees arrive in later
 Threads. No queue, background work, push, state framework, or new dependency was
 introduced.
 
+## M04: cached reads and explicit refresh state
+
+Startup still reads native SQLite before making any network request; refresh is
+always explicit. The screen now distinguishes stale, refreshing, fresh and error
+states from local-save errors. Fresh means checked at the last successful refresh,
+not continuous knowledge of another client's edits. A failed refresh keeps the
+saved list visible and does not advance that time. A local edit marks the display
+stale again; the M03 session still handles its later explicit synchronization.
+
+Schema v2 adds one `sync_metadata` row for the last successful-refresh timestamp.
+It commits in the same transaction as the canonical Item snapshot. Schema-v1
+upgrade adds only this metadata, preserving Items and the identity allocator.
+Cold startup is stale even when a prior refresh time exists, because no new
+network check has occurred. There is no age threshold, polling or automatic retry.
+
+The M04 Android harness controls actual OS connectivity, confirms no active
+network during offline reads, and restores the original settings. A debug-only
+initial prop supplies the two fixed successful-refresh timestamps; it does not
+replace RN fetch or native SQLite. The test fixture adds only one-shot GET503 and
+explicit remote-title controls. No native module or new dependency is needed.
+Persisting the refresh time does not preserve upload intent: M03's session-only
+edit comparison and first-pull replacement limits remain unchanged.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
@@ -77,6 +100,8 @@ cd ..
 python3 scripts/verify_m02.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m02
 # M03 harness owns/stops its port-18081 fixture. Use a new evidence directory.
 python3 scripts/verify_m03.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m03
+# M04 uses the same fixed emulator and owns/restores its fixture/connectivity.
+python3 scripts/verify_m04.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m04
 # To run the fixture separately for manual foreground sync:
 node fixture/server.cjs
 ```
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index 9a5a708..bc923c1 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -82,11 +82,11 @@ test.each([/^INSERT INTO items/, /^END/])('M02 failure at %s rolls back Item and
 test('M02 unsupported schema is rejected without recreating or deleting existing data', async () => {
   const store = await openItemStore();
   const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
-  connection().exec('PRAGMA user_version = 2');
+  connection().exec('PRAGMA user_version = 3');
   closeDatabases();
-  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 2');
+  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 3');
   expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
-  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(2);
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(3);
 });
 
 test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
@@ -104,3 +104,32 @@ test('M03 baseline: separate local databases cannot observe another instance wit
   expect(reopenedSecond).toEqual([]);
   console.info('M03_ISOLATION_BASELINE', JSON.stringify({first: reopenedFirst, second: reopenedSecond}));
 });
+
+test('M04 upgrades a literal M03 schema without touching cached Items or local identity', async () => {
+  const name = 'm04-v1.db';
+  const database = connection(name);
+  database.exec(`
+    CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL);
+    CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0));
+    INSERT INTO local_identity (singleton, next_id) VALUES (1, 3);
+    PRAGMA user_version = 1;
+  `);
+  const seeds = [
+    {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
+    {id: 'remote-002', title: 'Beta', completed: false, version: 1, updatedAt: 1700000000000},
+  ];
+  for (const item of seeds) {
+    database.prepare('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)')
+      .run(item.id, item.title, Number(item.completed), item.version, item.updatedAt);
+  }
+  const store = await openItemStore(name);
+  expect(await store.read()).toEqual(seeds);
+  expect(await store.readLastSuccessfulRefresh()).toBeNull();
+  expect(database.prepare('PRAGMA user_version').get()?.user_version).toBe(2);
+  expect(database.prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
+  closeDatabases();
+  const reopened = await openItemStore(name);
+  expect(await reopened.read()).toEqual(seeds);
+  expect(await reopened.readLastSuccessfulRefresh()).toBeNull();
+  expect(connection(name).prepare('SELECT next_id FROM local_identity WHERE singleton = 1').get()?.next_id).toBe(3);
+});
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
index 64f39bd..dfb98a2 100644
--- a/__tests__/sync.test.ts
+++ b/__tests__/sync.test.ts
@@ -116,3 +116,80 @@ test('M03 distinct production sessions do not use the fixed test identity namesp
     clock.mockRestore(); random.mockRestore();
   }
 });
+
+const m04Revised = {...seeds[0], title: 'Remote revised', version: 2, updatedAt: 1700000201000};
+
+async function control(path: string, body: unknown) {
+  const response = await request(url + path, {method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify(body)});
+  expect(response.status).toBe(200);
+  return response.json();
+}
+
+test('M04 frozen HTTP failure and reconnect retain persistent Items and successful-refresh time', async () => {
+  let offline = false;
+  const transport: JsonRequest = (address, options) => offline
+    ? Promise.reject(new TypeError('Network request failed')) : request(address, options);
+  const now = jest.fn(() => 1700000200000);
+  const store = await openItemStore('m04-read.db');
+  const sync = new ForegroundSync(store, url, transport, 'device', now);
+  expect(await store.read()).toEqual([]);
+  expect(await store.readLastSuccessfulRefresh()).toBeNull();
+  await sync.synchronize();
+  expect(await store.read()).toEqual(seeds);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000200000);
+
+  offline = true;
+  await expect(sync.synchronize()).rejects.toThrow('Network request failed');
+  expect(await store.read()).toEqual(seeds);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000200000);
+  expect(fixture.state().requests).toHaveLength(1);
+  expect(now).toHaveBeenCalledTimes(1);
+
+  offline = false;
+  expect(await control('/__get-failures', {count: 1})).toEqual({getFailures: 1, delayMs: 0});
+  await expect(sync.synchronize()).rejects.toThrow('GET /items failed (HTTP 503)');
+  expect(await store.read()).toEqual(seeds);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000200000);
+  expect(fixture.state().requests).toHaveLength(2);
+  expect(now).toHaveBeenCalledTimes(1);
+
+  offline = true;
+  expect(await control('/__remote-change', {id: 'remote-001', title: 'Remote revised', updatedAt: 1700000201000}))
+    .toEqual({item: m04Revised});
+  expect(await store.read()).toEqual(seeds);
+  expect(fixture.state().requests).toHaveLength(2);
+  offline = false;
+  expect(await control('/__get-failures', {count: 0})).toEqual({getFailures: 0, delayMs: 0});
+  now.mockReturnValue(1700000202000);
+  await sync.synchronize();
+  expect(await store.read()).toEqual([m04Revised, seeds[1]]);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000202000);
+  expect(now).toHaveBeenCalledTimes(2);
+  expect(fixture.state()).toEqual({items: [m04Revised, seeds[1]], nextTimestamp: 1700000100000, requests: [
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: seeds}},
+    {method: 'GET', path: '/items', body: null, status: 503, response: {error: 'Temporary GET failure'}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [m04Revised, seeds[1]]}},
+  ]});
+  console.info('M04_HTTP_RECONCILIATION', JSON.stringify(fixture.state()));
+  closeDatabases();
+  const reopened = await openItemStore('m04-read.db');
+  expect(await reopened.read()).toEqual([m04Revised, seeds[1]]);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000202000);
+});
+
+test('M04 a failed snapshot metadata write rolls back Items and the successful-refresh time together', async () => {
+  const store = await openItemStore();
+  const now = jest.fn(() => 1700000200000);
+  const sync = new ForegroundSync(store, url, request, 'device', now);
+  await sync.synchronize();
+  await control('/__remote-change', {id: 'remote-001', title: 'Remote revised', updatedAt: 1700000201000});
+  now.mockReturnValue(1700000202000);
+  failNextSql(/^UPDATE sync_metadata/);
+  await expect(sync.synchronize()).rejects.toThrow('injected persistence failure');
+  expect(await store.read()).toEqual(seeds);
+  expect(await store.readLastSuccessfulRefresh()).toBe(1700000200000);
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(seeds);
+  expect(await reopened.readLastSuccessfulRefresh()).toBe(1700000200000);
+});
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index 68801a9..812ad57 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -9,9 +9,16 @@ class MainActivity : ReactActivity() {
     override fun getMainComponentName(): String = "OfflineItemTracker"
     override fun createReactActivityDelegate(): ReactActivityDelegate =
         object : DefaultReactActivityDelegate(this, mainComponentName, false) {
-            override fun getLaunchOptions(): Bundle? =
-                if (BuildConfig.DEBUG && intent.getBooleanExtra("m03FixedIdentity", false)) {
-                    Bundle().apply { putString("testIdentityPrefix", "device") }
-                } else null
+            override fun getLaunchOptions(): Bundle? {
+                if (!BuildConfig.DEBUG) return null
+                return Bundle().apply {
+                    if (intent.getBooleanExtra("m03FixedIdentity", false)) {
+                        putString("testIdentityPrefix", "device")
+                    }
+                    if (intent.getBooleanExtra("m04FixedClock", false)) {
+                        putBoolean("testRefreshClock", true)
+                    }
+                }
+            }
         }
 }
diff --git a/scripts/verify_m02.py b/scripts/verify_m02.py
index 7579a4f..e516ce8 100644
--- a/scripts/verify_m02.py
+++ b/scripts/verify_m02.py
@@ -99,11 +99,11 @@ def main():
         assert path.exists(), "Native database file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as database:
             assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert database.execute("PRAGMA user_version").fetchone()[0] == 1
+            assert database.execute("PRAGMA user_version").fetchone()[0] == 2
             assert [column[1] for column in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in database.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid")]
-            state = {"schema_version": 1, "items": items,
+            state = {"schema_version": 2, "items": items,
                      "next_id": database.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(state, indent=2))
         return state
diff --git a/scripts/verify_m03.py b/scripts/verify_m03.py
index f345df3..29685c0 100644
--- a/scripts/verify_m03.py
+++ b/scripts/verify_m03.py
@@ -138,11 +138,11 @@ def main():
         assert path.exists(), "Native SQLite file missing"
         with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as db:
             assert db.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
-            assert db.execute("PRAGMA user_version").fetchone()[0] == 1
+            assert db.execute("PRAGMA user_version").fetchone()[0] == 2
             assert [column[1] for column in db.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
             items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
                      for row in db.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY id")]
-            value = {"items": items, "schema_version": 1,
+            value = {"items": items, "schema_version": 2,
                      "next_id": db.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
         (evidence / f"{name}.json").write_text(json.dumps(value, indent=2) + "\n")
         return value
diff --git a/src/App.tsx b/src/App.tsx
index 24a3c7e..aa8b6a5 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -4,17 +4,28 @@ import {Item} from './items';
 import {ItemMutation, ItemStore, openItemStore} from './itemStore';
 import {ForegroundSync, SyncSession} from './sync';
 
-const defaultSync = (store: ItemStore, identityPrefix?: string) => new ForegroundSync(store, undefined, undefined, identityPrefix);
+const defaultSync = (store: ItemStore, identityPrefix?: string, testRefreshClock = false) => {
+  // Android supplies this prop only behind BuildConfig.DEBUG. Real network and
+  // native SQLite are unchanged; only successful-refresh time is deterministic.
+  let next = 1700000200000;
+  const clock = testRefreshClock ? () => {const value = next; next += 2000; return value;} : undefined;
+  return new ForegroundSync(store, undefined, undefined, identityPrefix, clock);
+};
 
-export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix}: {
+type RefreshState = {status: 'stale' | 'refreshing' | 'fresh'} | {status: 'error'; message: string};
+
+export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix, testRefreshClock = false}: {
   openStore?: () => Promise<ItemStore>;
-  createSync?: (store: ItemStore, identityPrefix?: string) => SyncSession;
+  createSync?: (store: ItemStore, identityPrefix?: string, testRefreshClock?: boolean) => SyncSession;
   testIdentityPrefix?: string;
+  testRefreshClock?: boolean;
 }) {
   const [items, setItems] = useState<Item[]>([]);
   const [ready, setReady] = useState(false);
   const [busy, setBusy] = useState(true);
   const [error, setError] = useState<string | null>(null);
+  const [refresh, setRefresh] = useState<RefreshState>({status: 'stale'});
+  const [lastSuccessfulRefreshAt, setLastSuccessfulRefreshAt] = useState<number | null>(null);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
@@ -29,10 +40,13 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     setError(null);
     openStore().then(async opened => {
       const saved = await opened.read();
+      const lastRefresh = await opened.readLastSuccessfulRefresh();
       if (active) {
         store.current = opened;
-        sync.current = createSync(opened, testIdentityPrefix);
+        sync.current = createSync(opened, testIdentityPrefix, testRefreshClock);
         setItems(saved);
+        setLastSuccessfulRefreshAt(lastRefresh);
+        setRefresh({status: 'stale'});
         setReady(true);
       }
     }).catch(reason => {
@@ -41,7 +55,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
       if (active) {busyRef.current = false; setBusy(false);}
     });
     return () => {active = false;};
-  }, [openStore, openAttempt, createSync, testIdentityPrefix]);
+  }, [openStore, openAttempt, createSync, testIdentityPrefix, testRefreshClock]);
 
   async function mutate(action: ItemMutation): Promise<boolean> {
     if (!store.current || busyRef.current) {return false;}
@@ -50,6 +64,7 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     setError(null);
     try {
       setItems(await store.current.mutate(action, sync.current?.initialized ? sync.current.identityPrefix : undefined));
+      setRefresh({status: 'stale'});
       return true;
     } catch (reason) {
       setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);
@@ -64,12 +79,16 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     if (!store.current || !sync.current || busyRef.current) {return;}
     busyRef.current = true;
     setBusy(true);
-    setError(null);
+    setRefresh({status: 'refreshing'});
     try {
       await sync.current.synchronize();
-      setItems(await store.current.read());
+      const saved = await store.current.read();
+      const lastRefresh = await store.current.readLastSuccessfulRefresh();
+      setItems(saved);
+      setLastSuccessfulRefreshAt(lastRefresh);
+      setRefresh({status: 'fresh'});
     } catch (reason) {
-      setError(`Could not synchronize: ${reason instanceof Error ? reason.message : String(reason)}`);
+      setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});
     } finally {
       busyRef.current = false;
       setBusy(false);
@@ -94,10 +113,22 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     <SafeAreaView style={styles.screen}>
       <Text style={styles.heading}>Offline Item Tracker</Text>
       <Text accessibilityLabel={busy ? 'Local storage busy' : ready ? (error ? 'Local storage error' : 'Local storage ready') : 'Local storage unavailable'}>
-        {busy ? (ready ? 'Saving locally…' : 'Opening local database…') : ready ? (error ? 'Change not saved' : 'Saved locally') : 'Local database unavailable'}
+        {busy ? (ready ? (refresh.status === 'refreshing' ? 'Refreshing; saved Items remain available' : 'Saving locally…') : 'Opening local database…') : ready ? (error ? 'Change not saved' : 'Saved locally') : 'Local database unavailable'}
       </Text>
       {error !== null && <Text accessibilityRole="alert">{error}</Text>}
       {!ready && !busy && <Button title="Retry opening database" onPress={() => setOpenAttempt(value => value + 1)} />}
+      {ready && <>
+        <Text accessibilityLabel={`Sync status: ${refresh.status}`}>
+          {refresh.status === 'fresh' ? 'Fresh at last successful refresh'
+            : refresh.status === 'refreshing' ? 'Refreshing; showing saved Items'
+              : refresh.status === 'error' ? 'Refresh error; saved Items may be stale'
+                : 'Stale; showing saved Items until explicit refresh'}
+        </Text>
+        <Text accessibilityLabel={`Last successful refresh: ${lastSuccessfulRefreshAt ?? 'never'}`}>
+          Last successful refresh: {lastSuccessfulRefreshAt === null ? 'never' : new Date(lastSuccessfulRefreshAt).toISOString()}
+        </Text>
+        {refresh.status === 'error' && <Text accessibilityRole="alert">{refresh.message}</Text>}
+      </>}
       <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
       <Text>{sync.current?.initialized
         ? 'Sync sends this session's local edits, then pulls the remote snapshot.'
diff --git a/src/itemStore.ts b/src/itemStore.ts
index ba95b4c..a82dc6c 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -2,7 +2,7 @@ import SQLite, {SQLResultSet, SQLTransaction, WebsqlDatabase} from 'react-native
 import {Item, ItemAction, itemsReducer} from './items';
 
 export const DATABASE_NAME = 'items.db';
-export const SCHEMA_VERSION = 1;
+export const SCHEMA_VERSION = 2;
 
 export type ItemRow = {
   id: string;
@@ -33,8 +33,9 @@ export type ItemMutation = Exclude<ItemAction, {type: 'create'}>
 
 export interface ItemStore {
   read(): Promise<Item[]>;
+  readLastSuccessfulRefresh(): Promise<number | null>;
   mutate(action: ItemMutation, identityPrefix?: string): Promise<Item[]>;
-  replaceSnapshot(items: Item[]): Promise<void>;
+  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void>;
 }
 
 function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
@@ -59,7 +60,26 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
-  replaceSnapshot(items: Item[]): Promise<void> {
+  readLastSuccessfulRefresh(): Promise<number | null> {
+    return new Promise((resolve, reject) => {
+      let last: number | null = null;
+      this.database.readTransaction(tx => {
+        tx.executeSql('SELECT last_successful_refresh_at FROM sync_metadata WHERE singleton = 1', [], (_, result) => {
+          const value = result.rows.item(0)?.last_successful_refresh_at;
+          if (value !== null && (!Number.isSafeInteger(value) || value < 0)) {
+            throw new Error('Invalid last successful refresh time');
+          }
+          last = value;
+        });
+      }, reject, () => resolve(last));
+    });
+  }
+
+  replaceSnapshot(items: Item[], lastSuccessfulRefreshAt?: number): Promise<void> {
+    if (lastSuccessfulRefreshAt !== undefined
+        && (!Number.isSafeInteger(lastSuccessfulRefreshAt) || lastSuccessfulRefreshAt < 0)) {
+      return Promise.reject(new Error('Invalid successful refresh time'));
+    }
     return new Promise((resolve, reject) => {
       this.database.transaction(tx => {
         tx.executeSql('DELETE FROM items');
@@ -68,6 +88,12 @@ class SqliteItemStore implements ItemStore {
           tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
             [row.id, row.title, row.completed, row.version, row.updated_at]);
         }
+        if (lastSuccessfulRefreshAt !== undefined) {
+          // The recorded success describes this exact committed snapshot. A
+          // failed Item or metadata statement rolls back both in one transaction.
+          tx.executeSql('UPDATE sync_metadata SET last_successful_refresh_at = ? WHERE singleton = 1',
+            [lastSuccessfulRefreshAt]);
+        }
       }, reject, resolve);
     });
   }
@@ -132,6 +158,12 @@ export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
           tx.executeSql('CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL)');
           tx.executeSql('CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0))');
           tx.executeSql('INSERT INTO local_identity (singleton, next_id) VALUES (1, 1)');
+        }
+        if (version === 0 || version === 1) {
+          // M03 rows and identity allocation remain untouched during upgrade.
+          // An old database has no recorded successful-refresh timestamp.
+          tx.executeSql('CREATE TABLE sync_metadata (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), last_successful_refresh_at INTEGER CHECK(last_successful_refresh_at IS NULL OR last_successful_refresh_at >= 0))');
+          tx.executeSql('INSERT INTO sync_metadata (singleton, last_successful_refresh_at) VALUES (1, NULL)');
           tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
         } else if (version !== SCHEMA_VERSION) {
           throw new Error(`Unsupported local database schema ${version}`);
diff --git a/src/sync.ts b/src/sync.ts
index 6cb9a6b..857220d 100644
--- a/src/sync.ts
+++ b/src/sync.ts
@@ -38,7 +38,8 @@ export class ForegroundSync implements SyncSession {
   constructor(private readonly store: ItemStore,
     private readonly baseUrl = FIXTURE_URL,
     private readonly request: JsonRequest = (url, options) => fetch(url, options),
-    readonly identityPrefix = sessionIdentity()) {}
+    readonly identityPrefix = sessionIdentity(),
+    private readonly now: () => number = () => Date.now()) {}
 
   get initialized() {return this.baseline !== null;}
 
@@ -85,7 +86,7 @@ export class ForegroundSync implements SyncSession {
     if (new Set(snapshot.map(item => item.id)).size !== snapshot.length) {
       throw new Error('Duplicate Item in remote snapshot');
     }
-    await this.store.replaceSnapshot(snapshot);
+    await this.store.replaceSnapshot(snapshot, this.now());
     // Future comparisons and UI publication both come from committed SQLite.
     this.baseline = await this.store.read();
   }


