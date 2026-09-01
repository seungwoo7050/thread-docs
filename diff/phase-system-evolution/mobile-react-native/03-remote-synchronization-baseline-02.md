## `feat: add explicit foreground synchronization`

diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 1c39140..e3233af 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -81,3 +81,19 @@ test('M02 database opening error disables writes and offers a non-destructive re
   await saved();
   expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
 });
+
+test('M03 explicit sync publishes a committed database read, and never a service response', async () => {
+  const fixture = [{id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000}];
+  const store = await openItemStore();
+  const read = jest.spyOn(store, 'read');
+  const synchronize = jest.fn(async () => {await store.replaceSnapshot(fixture);});
+  render(<App openStore={async () => store} createSync={() => ({initialized: true, identityPrefix: 'device', synchronize})} />);
+  await saved();
+  expect(screen.queryByText('Alpha')).toBeNull();
+  fireEvent.press(screen.getByLabelText('Synchronize'));
+  await saved();
+  expect(synchronize).toHaveBeenCalledTimes(1);
+  expect(read).toHaveBeenCalledTimes(2);
+  expect(screen.getByText('Alpha')).toBeTruthy();
+  expect(screen.getByTestId('item-row-remote-001')).toBeTruthy();
+});
diff --git a/__tests__/sqliteNative.ts b/__tests__/sqliteNative.ts
index 5f2d800..ef0b813 100644
--- a/__tests__/sqliteNative.ts
+++ b/__tests__/sqliteNative.ts
@@ -29,7 +29,7 @@ export function resetSqlite() {
 
 export function cleanupSqlite() {
   closeDatabases();
-  rmSync(directory, {recursive: true});
+  if (directory) {rmSync(directory, {recursive: true});}
   directory = '';
 }
 
diff --git a/__tests__/sync.test.ts b/__tests__/sync.test.ts
new file mode 100644
index 0000000..64f39bd
--- /dev/null
+++ b/__tests__/sync.test.ts
@@ -0,0 +1,118 @@
+/// <reference types="node" />
+import {request as httpRequest, Server} from 'node:http';
+import {ForegroundSync, JsonRequest} from '../src/sync';
+import {openItemStore} from '../src/itemStore';
+import {closeDatabases, failNextSql} from './sqliteNative';
+
+type Trace = {method: string; path: string; body: unknown; status: number; response: unknown};
+const {createFixture} = require('../fixture/server.cjs') as {createFixture(): {
+  server: Server; reset(): void; state(): {items: unknown[]; nextTimestamp: number; requests: Trace[]};
+}};
+const fixture = createFixture();
+const url = 'http://127.0.0.1:18081';
+
+// The service uses RN fetch on Android. On the host this transport speaks real
+// loopback HTTP instead of Jest's RN XMLHttpRequest mock; it does not mock replies.
+const request: JsonRequest = (address, options) => new Promise((resolve, reject) => {
+  const outgoing = httpRequest(address, options, response => {
+    const chunks: Buffer[] = [];
+    response.on('data', chunk => chunks.push(chunk));
+    response.on('end', () => resolve({status: response.statusCode!,
+      json: async () => JSON.parse(Buffer.concat(chunks).toString('utf8'))}));
+  });
+  outgoing.on('error', reject);
+  outgoing.end(options.body);
+});
+
+beforeAll(() => new Promise<void>((resolve, reject) => {
+  fixture.server.once('error', reject);
+  fixture.server.listen(18081, '127.0.0.1', resolve);
+}));
+beforeEach(() => fixture.reset());
+afterAll(() => new Promise<void>(resolve => fixture.server.close(() => resolve())));
+
+const seeds = [
+  {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
+  {id: 'remote-002', title: 'Beta', completed: false, version: 1, updatedAt: 1700000000000},
+];
+const gamma = {id: 'device-001', title: 'Gamma', completed: false, version: 1, updatedAt: 1700000100000};
+const renamed = {...seeds[0], title: 'Alpha synced', version: 2, updatedAt: 1700000101000};
+const completed = {...renamed, completed: true, version: 3, updatedAt: 1700000102000};
+const final = [gamma, completed];
+
+test('M03 frozen HTTP contract synchronizes two independent persistent databases exactly', async () => {
+  const first = await openItemStore('m03-sync-first.db');
+  const sync = new ForegroundSync(first, url, request, 'device');
+  expect(await first.read()).toEqual([]);
+  await sync.synchronize();
+  expect(await first.read()).toEqual(seeds);
+  expect(sync.initialized).toBe(true);
+
+  await first.mutate({type: 'create', title: 'Gamma', now: 1700000100000}, sync.identityPrefix);
+  await sync.synchronize();
+  expect(await first.read()).toEqual([gamma, ...seeds]);
+  await first.mutate({type: 'rename', id: 'remote-001', title: 'Alpha synced', now: 1700000101000});
+  await sync.synchronize();
+  expect(await first.read()).toEqual([gamma, renamed, seeds[1]]);
+  await first.mutate({type: 'toggle', id: 'remote-001', now: 1700000102000});
+  await sync.synchronize();
+  expect(await first.read()).toEqual([gamma, completed, seeds[1]]);
+  await first.mutate({type: 'delete', id: 'remote-002'});
+  await sync.synchronize();
+  expect(await first.read()).toEqual(final);
+
+  const second = await openItemStore('m03-sync-second.db');
+  expect(await second.read()).toEqual([]);
+  await new ForegroundSync(second, url, request).synchronize();
+  expect(await second.read()).toEqual(final);
+  expect(fixture.state()).toEqual({items: final, nextTimestamp: 1700000104000, requests: [
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: seeds}},
+    {method: 'POST', path: '/items', body: {id: 'device-001', title: 'Gamma', completed: false}, status: 201, response: {item: gamma}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, ...seeds]}},
+    {method: 'PATCH', path: '/items/remote-001', body: {title: 'Alpha synced'}, status: 200, response: {item: renamed}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, renamed, seeds[1]]}},
+    {method: 'PATCH', path: '/items/remote-001', body: {completed: true}, status: 200, response: {item: completed}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: [gamma, completed, seeds[1]]}},
+    {method: 'DELETE', path: '/items/remote-002', body: null, status: 200, response: {deletedId: 'remote-002'}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final}},
+    {method: 'GET', path: '/items', body: null, status: 200, response: {items: final}},
+  ]});
+  console.info('M03_HTTP_CONVERGENCE', JSON.stringify(fixture.state()));
+  closeDatabases();
+  expect(await (await openItemStore('m03-sync-first.db')).read()).toEqual(final);
+  expect(await (await openItemStore('m03-sync-second.db')).read()).toEqual(final);
+});
+
+test('M03 snapshot replacement commits atomically and a failed local INSERT does not publish the pull', async () => {
+  const store = await openItemStore();
+  const original = await store.mutate({type: 'create', title: 'Local only', now: 1700000000000});
+  const sync = new ForegroundSync(store, url, request, 'device');
+  failNextSql(/^INSERT INTO items/);
+  await expect(sync.synchronize()).rejects.toThrow('injected persistence failure');
+  expect(sync.initialized).toBe(false);
+  closeDatabases();
+  expect(await (await openItemStore()).read()).toEqual(original);
+});
+
+test('M03 distinct production sessions do not use the fixed test identity namespace', async () => {
+  const clock = jest.spyOn(Date, 'now').mockReturnValue(1700000100000);
+  const random = jest.spyOn(Math, 'random').mockReturnValueOnce(0.125).mockReturnValueOnce(0.25)
+    .mockReturnValueOnce(0.5).mockReturnValueOnce(0.75);
+  try {
+    const first = await openItemStore('identity-first.db');
+    const second = await openItemStore('identity-second.db');
+    const a = new ForegroundSync(first, url, request);
+    const b = new ForegroundSync(second, url, request);
+    expect(a.identityPrefix).not.toBe(b.identityPrefix);
+    expect(a.identityPrefix).not.toBe('device');
+    expect(b.identityPrefix).not.toBe('device');
+    const one = await first.mutate({type: 'create', title: 'First', now: 1700000100000}, a.identityPrefix);
+    const two = await second.mutate({type: 'create', title: 'Second', now: 1700000100000}, b.identityPrefix);
+    expect(one[0].id).not.toBe(two[0].id);
+    closeDatabases();
+    expect(await (await openItemStore('identity-first.db')).read()).toEqual(one);
+    expect(await (await openItemStore('identity-second.db')).read()).toEqual(two);
+  } finally {
+    clock.mockRestore(); random.mockRestore();
+  }
+});
diff --git a/android/app/src/main/AndroidManifest.xml b/android/app/src/main/AndroidManifest.xml
index d62dc17..002b0cf 100644
--- a/android/app/src/main/AndroidManifest.xml
+++ b/android/app/src/main/AndroidManifest.xml
@@ -1,5 +1,6 @@
 <manifest xmlns:android="http://schemas.android.com/apk/res/android">
-    <application android:name=".MainApplication" android:label="Offline Item Tracker" android:allowBackup="false" android:theme="@style/AppTheme" android:supportsRtl="true">
+    <uses-permission android:name="android.permission.INTERNET" />
+    <application android:name=".MainApplication" android:label="Offline Item Tracker" android:allowBackup="false" android:theme="@style/AppTheme" android:supportsRtl="true" android:networkSecurityConfig="@xml/network_security_config">
         <activity android:name=".MainActivity" android:exported="true" android:windowSoftInputMode="adjustResize">
             <intent-filter>
                 <action android:name="android.intent.action.MAIN" />
diff --git a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
index ac5e87f..68801a9 100644
--- a/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
+++ b/android/app/src/main/java/com/mse/reactnative/MainActivity.kt
@@ -1,5 +1,6 @@
 package com.mse.reactnative
 
+import android.os.Bundle
 import com.facebook.react.ReactActivity
 import com.facebook.react.ReactActivityDelegate
 import com.facebook.react.defaults.DefaultReactActivityDelegate
@@ -7,5 +8,10 @@ import com.facebook.react.defaults.DefaultReactActivityDelegate
 class MainActivity : ReactActivity() {
     override fun getMainComponentName(): String = "OfflineItemTracker"
     override fun createReactActivityDelegate(): ReactActivityDelegate =
-        DefaultReactActivityDelegate(this, mainComponentName, false)
+        object : DefaultReactActivityDelegate(this, mainComponentName, false) {
+            override fun getLaunchOptions(): Bundle? =
+                if (BuildConfig.DEBUG && intent.getBooleanExtra("m03FixedIdentity", false)) {
+                    Bundle().apply { putString("testIdentityPrefix", "device") }
+                } else null
+        }
 }
diff --git a/android/app/src/main/res/xml/network_security_config.xml b/android/app/src/main/res/xml/network_security_config.xml
new file mode 100644
index 0000000..70d48a3
--- /dev/null
+++ b/android/app/src/main/res/xml/network_security_config.xml
@@ -0,0 +1,7 @@
+<?xml version="1.0" encoding="utf-8"?>
+<network-security-config>
+    <base-config cleartextTrafficPermitted="false" />
+    <domain-config cleartextTrafficPermitted="true">
+        <domain>10.0.2.2</domain>
+    </domain-config>
+</network-security-config>
diff --git a/fixture/server.cjs b/fixture/server.cjs
new file mode 100644
index 0000000..d25f476
--- /dev/null
+++ b/fixture/server.cjs
@@ -0,0 +1,86 @@
+// A disposable deterministic M03 fixture, not a backend service.
+const http = require('node:http');
+
+const SEEDS = [
+  {id: 'remote-001', title: 'Alpha', completed: false, version: 1, updatedAt: 1700000000000},
+  {id: 'remote-002', title: 'Beta', completed: false, version: 1, updatedAt: 1700000000000},
+];
+
+function createFixture() {
+  let items;
+  let nextTimestamp;
+  let requests;
+  const snapshot = () => [...items.values()].sort((a, b) => a.id.localeCompare(b.id));
+  function reset() {
+    items = new Map(SEEDS.map(item => [item.id, {...item}]));
+    nextTimestamp = 1700000100000;
+    requests = [];
+  }
+  reset();
+  const state = () => ({items: snapshot(), nextTimestamp, requests});
+  const tick = () => {const timestamp = nextTimestamp; nextTimestamp += 1000; return timestamp;};
+
+  const server = http.createServer(async (request, response) => {
+    let body;
+    const {method, url: path} = request;
+    function reply(status, payload) {
+      if (!path.startsWith('/__')) {requests.push({method, path, body: body ?? null, status, response: payload});}
+      response.writeHead(status, {'Content-Type': 'application/json'});
+      response.end(JSON.stringify(payload));
+    }
+    try {
+      const chunks = [];
+      for await (const chunk of request) {chunks.push(chunk);}
+      const raw = Buffer.concat(chunks).toString('utf8');
+      if (raw) {body = JSON.parse(raw);}
+      if (method === 'POST' && path === '/__reset') {reset(); return reply(200, state());}
+      if (method === 'GET' && path === '/__state') {return reply(200, state());}
+      if (method === 'GET' && path === '/items') {return reply(200, {items: snapshot()});}
+      if (method === 'POST' && path === '/items') {
+        if (!body || typeof body.id !== 'string' || !/^[a-zA-Z0-9_-]+$/.test(body.id)
+            || typeof body.title !== 'string' || !body.title.trim() || typeof body.completed !== 'boolean') {
+          return reply(400, {error: 'id, title, and completed are required'});
+        }
+        if (items.has(body.id)) {return reply(409, {error: 'Item already exists'});}
+        const item = {id: body.id, title: body.title.trim(), completed: body.completed, version: 1, updatedAt: tick()};
+        items.set(item.id, item);
+        return reply(201, {item});
+      }
+      const match = /^\/items\/([a-zA-Z0-9_-]+)$/.exec(path);
+      if (match && (method === 'PATCH' || method === 'DELETE')) {
+        const item = items.get(match[1]);
+        if (!item) {return reply(404, {error: 'Item not found'});}
+        if (method === 'DELETE') {
+          items.delete(item.id);
+          tick();
+          return reply(200, {deletedId: item.id});
+        }
+        if (!body || !Object.keys(body).length
+            || Object.keys(body).some(key => key !== 'title' && key !== 'completed')
+            || ('title' in body && (typeof body.title !== 'string' || !body.title.trim()))
+            || ('completed' in body && typeof body.completed !== 'boolean')) {
+          return reply(400, {error: 'Changed title or completed is required'});
+        }
+        const updated = {...item, ...body, ...(body.title === undefined ? {} : {title: body.title.trim()}),
+          version: item.version + 1, updatedAt: tick()};
+        items.set(item.id, updated);
+        return reply(200, {item: updated});
+      }
+      return reply(404, {error: 'Not found'});
+    } catch {
+      return reply(400, {error: 'Invalid JSON request'});
+    }
+  });
+  return {server, reset, state};
+}
+
+module.exports = {createFixture};
+if (require.main === module) {
+  const fixture = createFixture();
+  fixture.server.listen(18081, '127.0.0.1', () => {
+    console.log(JSON.stringify({pid: process.pid, host: '127.0.0.1', port: 18081}));
+  });
+  for (const signal of ['SIGINT', 'SIGTERM']) {
+    process.on(signal, () => fixture.server.close(() => process.exit(0)));
+  }
+}
diff --git a/src/App.tsx b/src/App.tsx
index 36ffa7b..24a3c7e 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -2,14 +2,22 @@ import React, {useEffect, useRef, useState} from 'react';
 import {Button, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
 import {Item} from './items';
 import {ItemMutation, ItemStore, openItemStore} from './itemStore';
+import {ForegroundSync, SyncSession} from './sync';
 
-export default function App({openStore = openItemStore}: {openStore?: () => Promise<ItemStore>}) {
+const defaultSync = (store: ItemStore, identityPrefix?: string) => new ForegroundSync(store, undefined, undefined, identityPrefix);
+
+export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix}: {
+  openStore?: () => Promise<ItemStore>;
+  createSync?: (store: ItemStore, identityPrefix?: string) => SyncSession;
+  testIdentityPrefix?: string;
+}) {
   const [items, setItems] = useState<Item[]>([]);
   const [ready, setReady] = useState(false);
   const [busy, setBusy] = useState(true);
   const [error, setError] = useState<string | null>(null);
   const [openAttempt, setOpenAttempt] = useState(0);
   const store = useRef<ItemStore | null>(null);
+  const sync = useRef<SyncSession | null>(null);
   const busyRef = useRef(true);
   const [draft, setDraft] = useState('');
   const [editingId, setEditingId] = useState<string | null>(null);
@@ -23,6 +31,7 @@ export default function App({openStore = openItemStore}: {openStore?: () => Prom
       const saved = await opened.read();
       if (active) {
         store.current = opened;
+        sync.current = createSync(opened, testIdentityPrefix);
         setItems(saved);
         setReady(true);
       }
@@ -32,7 +41,7 @@ export default function App({openStore = openItemStore}: {openStore?: () => Prom
       if (active) {busyRef.current = false; setBusy(false);}
     });
     return () => {active = false;};
-  }, [openStore, openAttempt]);
+  }, [openStore, openAttempt, createSync, testIdentityPrefix]);
 
   async function mutate(action: ItemMutation): Promise<boolean> {
     if (!store.current || busyRef.current) {return false;}
@@ -40,7 +49,7 @@ export default function App({openStore = openItemStore}: {openStore?: () => Prom
     setBusy(true);
     setError(null);
     try {
-      setItems(await store.current.mutate(action));
+      setItems(await store.current.mutate(action, sync.current?.initialized ? sync.current.identityPrefix : undefined));
       return true;
     } catch (reason) {
       setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);
@@ -51,6 +60,22 @@ export default function App({openStore = openItemStore}: {openStore?: () => Prom
     }
   }
 
+  async function synchronize() {
+    if (!store.current || !sync.current || busyRef.current) {return;}
+    busyRef.current = true;
+    setBusy(true);
+    setError(null);
+    try {
+      await sync.current.synchronize();
+      setItems(await store.current.read());
+    } catch (reason) {
+      setError(`Could not synchronize: ${reason instanceof Error ? reason.message : String(reason)}`);
+    } finally {
+      busyRef.current = false;
+      setBusy(false);
+    }
+  }
+
   async function saveTitle() {
     if (!draft.trim()) {
       return;
@@ -73,6 +98,10 @@ export default function App({openStore = openItemStore}: {openStore?: () => Prom
       </Text>
       {error !== null && <Text accessibilityRole="alert">{error}</Text>}
       {!ready && !busy && <Button title="Retry opening database" onPress={() => setOpenAttempt(value => value + 1)} />}
+      <Button title="Synchronize" accessibilityLabel="Synchronize" disabled={!ready || busy} onPress={synchronize} />
+      <Text>{sync.current?.initialized
+        ? 'Sync sends this session's local edits, then pulls the remote snapshot.'
+        : 'First sync pulls the remote snapshot, replacing local Items. Sync edits before closing this session.'}</Text>
       <TextInput
         accessibilityLabel={editingId === null ? 'New item title' : 'Edit item title'}
         placeholder="Item title"
diff --git a/src/itemStore.ts b/src/itemStore.ts
index f2bd259..ba95b4c 100644
--- a/src/itemStore.ts
+++ b/src/itemStore.ts
@@ -33,7 +33,8 @@ export type ItemMutation = Exclude<ItemAction, {type: 'create'}>
 
 export interface ItemStore {
   read(): Promise<Item[]>;
-  mutate(action: ItemMutation): Promise<Item[]>;
+  mutate(action: ItemMutation, identityPrefix?: string): Promise<Item[]>;
+  replaceSnapshot(items: Item[]): Promise<void>;
 }
 
 function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
@@ -58,7 +59,20 @@ class SqliteItemStore implements ItemStore {
     });
   }
 
-  mutate(action: ItemMutation): Promise<Item[]> {
+  replaceSnapshot(items: Item[]): Promise<void> {
+    return new Promise((resolve, reject) => {
+      this.database.transaction(tx => {
+        tx.executeSql('DELETE FROM items');
+        for (const item of items) {
+          const row = itemToRow(item);
+          tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+            [row.id, row.title, row.completed, row.version, row.updated_at]);
+        }
+      }, reject, resolve);
+    });
+  }
+
+  mutate(action: ItemMutation, identityPrefix = 'item'): Promise<Item[]> {
     return new Promise((resolve, reject) => {
       let committed: Item[] = [];
       this.database.transaction(tx => {
@@ -89,7 +103,7 @@ class SqliteItemStore implements ItemStore {
               if (!Number.isSafeInteger(nextId) || nextId < 1) {
                 throw new Error('Invalid local Item identity sequence');
               }
-              const id = `item-${String(nextId).padStart(3, '0')}`;
+              const id = `${identityPrefix}-${String(nextId).padStart(3, '0')}`;
               if (current.some(item => item.id === id)) {
                 throw new Error('Local Item identity already exists');
               }
diff --git a/src/sync.ts b/src/sync.ts
new file mode 100644
index 0000000..6cb9a6b
--- /dev/null
+++ b/src/sync.ts
@@ -0,0 +1,92 @@
+import {Item} from './items';
+import {ItemStore} from './itemStore';
+
+export const FIXTURE_URL = 'http://10.0.2.2:18081';
+
+export type JsonRequest = (url: string, options: {
+  method: string; headers: Record<string, string>; body?: string;
+}) => Promise<{status: number; json(): Promise<unknown>}>;
+
+export interface SyncSession {
+  readonly initialized: boolean;
+  readonly identityPrefix: string;
+  synchronize(): Promise<void>;
+}
+
+function remoteItem(value: unknown): Item {
+  const item = value as Item;
+  if (!item || typeof item.id !== 'string' || !item.id
+      || typeof item.title !== 'string' || !item.title.trim()
+      || typeof item.completed !== 'boolean'
+      || !Number.isSafeInteger(item.version) || item.version < 1
+      || !Number.isSafeInteger(item.updatedAt)) {
+    throw new Error('Invalid remote Item');
+  }
+  return {id: item.id, title: item.title, completed: item.completed,
+    version: item.version, updatedAt: item.updatedAt};
+}
+
+// Item IDs need distinct namespaces as soon as devices share remote state.
+// Full IDs persist in SQLite. This nonce is not a clientMutationId protocol.
+function sessionIdentity() {
+  return `device-${Date.now().toString(36)}-${Math.random().toString(36).slice(2)}-${Math.random().toString(36).slice(2)}`;
+}
+
+export class ForegroundSync implements SyncSession {
+  private baseline: Item[] | null = null;
+
+  constructor(private readonly store: ItemStore,
+    private readonly baseUrl = FIXTURE_URL,
+    private readonly request: JsonRequest = (url, options) => fetch(url, options),
+    readonly identityPrefix = sessionIdentity()) {}
+
+  get initialized() {return this.baseline !== null;}
+
+  private async exchange(method: string, path: string, expectedStatus: number, body?: unknown) {
+    const response = await this.request(`${this.baseUrl}${path}`, {
+      method, headers: {'Content-Type': 'application/json'},
+      ...(body === undefined ? {} : {body: JSON.stringify(body)}),
+    });
+    if (response.status !== expectedStatus) {
+      throw new Error(`${method} ${path} failed (HTTP ${response.status})`);
+    }
+    return response.json();
+  }
+
+  async synchronize(): Promise<void> {
+    // M03 remembers only this foreground session's last shared snapshot, not
+    // durable pending intent. The first explicit sync is a remote pull.
+    if (this.baseline !== null) {
+      const local = await this.store.read();
+      const previous = new Map(this.baseline.map(item => [item.id, item]));
+      for (const item of [...local].sort((a, b) => a.id.localeCompare(b.id))) {
+        const before = previous.get(item.id);
+        if (!before) {
+          await this.exchange('POST', '/items', 201, {id: item.id, title: item.title, completed: item.completed});
+        } else {
+          const patch: {title?: string; completed?: boolean} = {};
+          if (item.title !== before.title) {patch.title = item.title;}
+          if (item.completed !== before.completed) {patch.completed = item.completed;}
+          if (Object.keys(patch).length) {
+            await this.exchange('PATCH', `/items/${encodeURIComponent(item.id)}`, 200, patch);
+          }
+        }
+      }
+      for (const item of [...this.baseline].sort((a, b) => a.id.localeCompare(b.id))) {
+        if (!local.some(value => value.id === item.id)) {
+          await this.exchange('DELETE', `/items/${encodeURIComponent(item.id)}`, 200);
+        }
+      }
+    }
+
+    const response = await this.exchange('GET', '/items', 200) as {items?: unknown};
+    if (!Array.isArray(response.items)) {throw new Error('Invalid remote snapshot');}
+    const snapshot = response.items.map(remoteItem);
+    if (new Set(snapshot.map(item => item.id)).size !== snapshot.length) {
+      throw new Error('Duplicate Item in remote snapshot');
+    }
+    await this.store.replaceSnapshot(snapshot);
+    // Future comparisons and UI publication both come from committed SQLite.
+    this.baseline = await this.store.read();
+  }
+}


