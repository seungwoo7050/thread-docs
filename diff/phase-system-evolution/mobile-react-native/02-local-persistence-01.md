# M02 — Local Persistence

## `feat: preserve Items across Android process restarts`

diff --git a/.gitignore b/.gitignore
index edc72f1..28e9e77 100644
--- a/.gitignore
+++ b/.gitignore
@@ -9,3 +9,4 @@ android/local.properties
 android/.kotlin/
 android/app/.cxx/
 *.keystore
+__pycache__/
diff --git a/TRACK.md b/TRACK.md
index 91e31c1..8dabfb4 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -3,7 +3,7 @@
 Independent Android-only TypeScript/React Native implementation of Mobile Systems Evolution.
 The immutable specification revision is in `SPEC_REVISION`; every commit identifies its Thread.
 
-## M01: process memory
+## M01: process memory (historical baseline)
 
 One screen creates, renames, completes, uncompletes, and deletes Items. React owns the
 in-memory Item array. Item IDs remain stable when editing; `version` is a local edit
@@ -11,19 +11,40 @@ counter without synchronization meaning. Nothing is persisted, and a cold proces
 restart clears all Items. There is no network client, database, sync, background work,
 custom native module, or state-management library.
 
+## M02: local persistence
+
+The same screen now reads and mutates Items through native SQLite using
+`react-native-sqlite-2` 3.6.3. Schema `user_version=1` is created atomically on first
+open. Existing databases are opened without deleting data; unknown schema versions
+fail explicitly. The Item table contains only the five domain fields. A separate
+local identity counter commits with each new Item, so deleted IDs are not reused
+when the application restarts.
+
+The UI shows committed database snapshots. A statement or commit failure leaves
+the prior list and draft visible and reports that the change was not saved.
+No network client, sync state, pending queue, background work, or application
+native module is added. The dependency patch only supplies its AGP namespace and
+removes a legacy manifest namespace and unused null NDK-version setting.
+
 ## Toolchain and commands
 
-Use Node 22, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
+Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
 API 34 Pixel 6 emulator. Set `JAVA_HOME`, `ANDROID_HOME`, and `ANDROID_SERIAL` for the
-local machine; no machine paths are checked in. The Gradle wrapper pins Gradle 8.10.2.
+local machine. Build configuration contains no machine paths. The Gradle wrapper pins Gradle 8.10.2.
 
 ```sh
-npm ci
+npm ci --ignore-scripts
+npm run postinstall
 npm run typecheck
 npm test -- --runInBand
 cd android
 ./gradlew :app:assembleDebug :app:assembleDebugAndroidTest
+# The M01 fixture begins with an empty installation on each run.
+adb shell pm clear com.mse.reactnative
 ./gradlew :app:connectedDebugAndroidTest
+cd ..
+# With the shared device lease granted:
+python3 scripts/verify_m02.py --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /tmp/mse-rn-m02
 ```
 
 Debug builds bundle their JavaScript and use Hermes with developer support disabled,
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index 119a013..1c39140 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -1,30 +1,42 @@
 import React from 'react';
-import {fireEvent, render, screen} from '@testing-library/react-native';
+import {fireEvent, render, screen, waitFor} from '@testing-library/react-native';
 import App from '../src/App';
+import {openItemStore} from '../src/itemStore';
+import {closeDatabases, failNextSql} from './sqliteNative';
 
-test('M01 fixed sequence maps stable Item identity to the rendered list', () => {
+const saved = () => waitFor(() => expect(screen.getByLabelText('Local storage ready')).toBeTruthy());
+
+test('M01 fixed sequence maps stable Item identity to the rendered list', async () => {
   let clock = 1700000000000;
   const clockSpy = jest.spyOn(Date, 'now').mockImplementation(() => {const value = clock; clock += 1000; return value;});
   try {
     render(<App />);
+    await saved();
     expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
     fireEvent.changeText(screen.getByLabelText('New item title'), 'Alpha');
     fireEvent.press(screen.getByLabelText('Add item'));
+    await saved();
     fireEvent.changeText(screen.getByLabelText('New item title'), 'Beta');
     fireEvent.press(screen.getByLabelText('Add item'));
+    await saved();
     expect(screen.getByLabelText('Item count: 2')).toBeTruthy();
 
     fireEvent.press(screen.getByLabelText('Edit Alpha'));
     fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Alpha edited');
     fireEvent.press(screen.getByLabelText('Save title'));
+    await saved();
     expect(screen.getByTestId('item-title-item-001').props.children).toBe('Alpha edited');
 
     fireEvent.press(screen.getByLabelText('Mark Alpha edited complete'));
+    await saved();
     expect(screen.getByRole('checkbox', {name: 'Mark Alpha edited incomplete'}).props.accessibilityState.checked).toBe(true);
     fireEvent.press(screen.getByLabelText('Mark Alpha edited incomplete'));
+    await saved();
     expect(screen.getByRole('checkbox', {name: 'Mark Alpha edited complete'}).props.accessibilityState.checked).toBe(false);
     fireEvent.press(screen.getByLabelText('Mark Alpha edited complete'));
+    await saved();
     fireEvent.press(screen.getByLabelText('Delete Beta'));
+    await saved();
 
     expect(screen.getByLabelText('Item count: 1')).toBeTruthy();
     expect(screen.getByTestId('item-row-item-001')).toBeTruthy();
@@ -36,3 +48,36 @@ test('M01 fixed sequence maps stable Item identity to the rendered list', () =>
     clockSpy.mockRestore();
   }
 });
+
+test('M02 startup reads saved Items; failed commits leave the draft and confirmed UI unchanged', async () => {
+  const store = await openItemStore();
+  const original = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
+  render(<App />);
+  await saved();
+  expect(screen.getByText('Alpha')).toBeTruthy();
+  failNextSql(/^END/);
+  fireEvent.changeText(screen.getByLabelText('New item title'), 'Beta');
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await waitFor(() => expect(screen.getByRole('alert').props.children).toContain('injected persistence failure'));
+  expect(screen.getByText('Change not saved')).toBeTruthy();
+  expect(screen.queryByText('Saved locally')).toBeNull();
+  expect(screen.getByLabelText('Item count: 1')).toBeTruthy();
+  expect(screen.getByLabelText('New item title').props.value).toBe('Beta');
+  expect(screen.queryByTestId('item-row-item-002')).toBeNull();
+  closeDatabases();
+  expect(await (await openItemStore()).read()).toEqual(original);
+  fireEvent.press(screen.getByLabelText('Add item'));
+  await saved();
+  expect(screen.getByTestId('item-row-item-002')).toBeTruthy();
+});
+
+test('M02 database opening error disables writes and offers a non-destructive retry', async () => {
+  failNextSql(/^PRAGMA user_version/);
+  render(<App />);
+  await waitFor(() => expect(screen.getByRole('alert')).toBeTruthy());
+  expect(screen.getByLabelText('Add item').props.accessibilityState.disabled).toBe(true);
+  expect(screen.queryByLabelText('Item count: 0')).toBeNull();
+  fireEvent.press(screen.getByRole('button', {name: /retry opening database/i}));
+  await saved();
+  expect(screen.getByLabelText('Item count: 0')).toBeTruthy();
+});
diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index 5da8985..ce85aad 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -1,4 +1,7 @@
 import {Item, itemsReducer} from '../src/items';
+import SQLite from 'react-native-sqlite-2';
+import {itemToRow, openItemStore, rowToItem} from '../src/itemStore';
+import {closeDatabases, connection, failNextSql} from './sqliteNative';
 
 test('M01 fixed sequence preserves first identity and all Item fields', () => {
   let items: Item[] = [];
@@ -26,3 +29,62 @@ test('blank titles cannot create or erase an Item', () => {
   const items = itemsReducer([], {type: 'create', id: 'item-001', title: 'Alpha', now: 1700000000000});
   expect(itemsReducer(items, {type: 'rename', id: 'item-001', title: ' ', now: 1700000001000})).toEqual(items);
 });
+
+test('M02 fixed version-zero fixture maps all five fields through SQLite and database reopening', async () => {
+  const fixture: Item = {id: 'item-001', title: 'Alpha edited', completed: true, version: 0, updatedAt: 1700000006000};
+  const row = itemToRow(fixture);
+  expect(row).toEqual({id: 'item-001', title: 'Alpha edited', completed: 1, version: 0, updated_at: 1700000006000});
+  expect(rowToItem(row)).toEqual(fixture);
+  expect(rowToItem({...row, completed: 0}).completed).toBe(false);
+  await openItemStore();
+  const database = SQLite.openDatabase('items.db');
+  await new Promise<void>((resolve, reject) => database.transaction(tx => {
+    tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+      [row.id, row.title, row.completed, row.version, row.updated_at]);
+    tx.executeSql('UPDATE local_identity SET next_id = 2 WHERE singleton = 1');
+  }, reject, resolve));
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual([fixture]);
+});
+
+test('M02 database preserves the fixed M01 sequence and never reuses IDs after restart', async () => {
+  let store = await openItemStore();
+  let clock = 1700000000000;
+  const now = () => {const value = clock; clock += 1000; return value;};
+  expect(await store.read()).toEqual([]);
+  await store.mutate({type: 'create', title: 'Alpha', now: now()});
+  await store.mutate({type: 'create', title: 'Beta', now: now()});
+  await store.mutate({type: 'rename', id: 'item-001', title: 'Alpha edited', now: now()});
+  expect((await store.mutate({type: 'toggle', id: 'item-001', now: now()}))[0].completed).toBe(true);
+  expect((await store.mutate({type: 'toggle', id: 'item-001', now: now()}))[0].completed).toBe(false);
+  await store.mutate({type: 'toggle', id: 'item-001', now: now()});
+  const final = await store.mutate({type: 'delete', id: 'item-002'});
+  expect(final).toEqual([{id: 'item-001', title: 'Alpha edited', completed: true, version: 5, updatedAt: 1700000005000}]);
+  closeDatabases();
+  store = await openItemStore();
+  expect(await store.read()).toEqual(final);
+  const created = await store.mutate({type: 'create', title: 'Gamma', now: now()});
+  expect(created).toEqual([...final, {id: 'item-003', title: 'Gamma', completed: false, version: 1, updatedAt: 1700000006000}]);
+});
+
+test.each([/^INSERT INTO items/, /^END/])('M02 failure at %s rolls back Item and identity allocation', async failAt => {
+  const store = await openItemStore();
+  const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
+  failNextSql(failAt);
+  await expect(store.mutate({type: 'create', title: 'Beta', now: 1700000001000})).rejects.toThrow('injected persistence failure');
+  closeDatabases();
+  const reopened = await openItemStore();
+  expect(await reopened.read()).toEqual(saved);
+  expect((await reopened.mutate({type: 'create', title: 'Beta', now: 1700000001000}))[1].id).toBe('item-002');
+});
+
+test('M02 unsupported schema is rejected without recreating or deleting existing data', async () => {
+  const store = await openItemStore();
+  const saved = await store.mutate({type: 'create', title: 'Alpha', now: 1700000000000});
+  connection().exec('PRAGMA user_version = 2');
+  closeDatabases();
+  await expect(openItemStore()).rejects.toThrow('Unsupported local database schema 2');
+  expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
+  expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(2);
+});
diff --git a/__tests__/sqlite.setup.ts b/__tests__/sqlite.setup.ts
new file mode 100644
index 0000000..0b903b2
--- /dev/null
+++ b/__tests__/sqlite.setup.ts
@@ -0,0 +1,13 @@
+import {NativeModules} from 'react-native';
+import {cleanupSqlite, nativeSqlite, resetSqlite} from './sqliteNative';
+
+// This track targets Android; use its Button/native-control branch in host tests.
+jest.mock('react-native/Libraries/Utilities/Platform', () => ({
+  OS: 'android',
+  Version: 34,
+  select: (values: Record<string, unknown>) => values.android ?? values.native ?? values.default,
+}));
+
+NativeModules.RNSqlite2 = nativeSqlite;
+beforeEach(resetSqlite);
+afterEach(cleanupSqlite);
diff --git a/__tests__/sqliteNative.ts b/__tests__/sqliteNative.ts
new file mode 100644
index 0000000..5f2d800
--- /dev/null
+++ b/__tests__/sqliteNative.ts
@@ -0,0 +1,60 @@
+/// <reference types="node" />
+// Only the native bridge is replaced on the host. The production SQLite library's
+// JavaScript transactions and the application's SQL run against an actual file.
+import {mkdtempSync, rmSync} from 'node:fs';
+import {tmpdir} from 'node:os';
+import {join} from 'node:path';
+import {DatabaseSync, SQLInputValue} from 'node:sqlite';
+
+let directory = '';
+const databases = new Map<string, DatabaseSync>();
+let failure: RegExp | null = null;
+
+export function connection(name = 'items.db') {
+  if (!databases.has(name)) {databases.set(name, new DatabaseSync(join(directory, name)));}
+  return databases.get(name)!;
+}
+
+export function closeDatabases() {
+  for (const database of databases.values()) {database.close();}
+  databases.clear();
+}
+
+export function resetSqlite() {
+  closeDatabases();
+  if (directory) {rmSync(directory, {recursive: true});}
+  directory = mkdtempSync(join(tmpdir(), 'mse-rn-m02-'));
+  failure = null;
+}
+
+export function cleanupSqlite() {
+  closeDatabases();
+  rmSync(directory, {recursive: true});
+  directory = '';
+}
+
+export function failNextSql(pattern: RegExp) {failure = pattern;}
+
+export const nativeSqlite = {
+  async exec(name: string, queries: [string, SQLInputValue[]][], _readOnly: boolean) {
+    const database = connection(name);
+    return queries.map(([sql, parameters]) => {
+      try {
+        if (failure?.test(sql)) {
+          failure = null;
+          throw new Error('injected persistence failure');
+        }
+        const statement = database.prepare(sql);
+        const columns = statement.columns().map((column: {name: string}) => column.name);
+        if (columns.length) {
+          const rows = statement.all(...parameters);
+          return [null, 0, 0, columns, rows.map((row: Record<string, unknown>) => columns.map((column: string) => row[column]))];
+        }
+        const result = statement.run(...parameters);
+        return [null, Number(result.lastInsertRowid), Number(result.changes), [], []];
+      } catch (error) {
+        return [error instanceof Error ? error.message : String(error), 0, 0, [], []];
+      }
+    });
+  },
+};
diff --git a/jest.config.js b/jest.config.js
index 95420d2..1263c3e 100644
--- a/jest.config.js
+++ b/jest.config.js
@@ -1,4 +1,5 @@
 module.exports = {
   preset: 'react-native',
+  setupFilesAfterEnv: ['<rootDir>/__tests__/sqlite.setup.ts'],
   testMatch: ['**/__tests__/**/*.test.ts', '**/__tests__/**/*.test.tsx'],
 };
diff --git a/package-lock.json b/package-lock.json
index 1c2067d..14f9486 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -9,7 +9,8 @@
       "version": "0.1.0",
       "dependencies": {
         "react": "18.3.1",
-        "react-native": "0.76.9"
+        "react-native": "0.76.9",
+        "react-native-sqlite-2": "3.6.3"
       },
       "devDependencies": {
         "@babel/core": "7.25.2",
@@ -25,6 +26,7 @@
         "@types/react-test-renderer": "18.3.0",
         "babel-jest": "29.7.0",
         "jest": "29.7.0",
+        "patch-package": "8.0.0",
         "react-test-renderer": "18.3.1",
         "typescript": "5.6.3"
       },
@@ -2096,6 +2098,13 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/@gar/promisify": {
+      "version": "1.1.3",
+      "resolved": "https://registry.npmjs.org/@gar/promisify/-/promisify-1.1.3.tgz",
+      "integrity": "sha512-k2Ty1JcVojjJFwrg/ThKi2ujJ7XNLYaFGNB/bWT9wGR+oSMJHMa5w+CUq6p/pVrKeNNgA7pCqEcjSnHVoqJQFw==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/@hapi/hoek": {
       "version": "9.3.0",
       "resolved": "https://registry.npmjs.org/@hapi/hoek/-/hoek-9.3.0.tgz",
@@ -2672,6 +2681,58 @@
         "node": ">= 8"
       }
     },
+    "node_modules/@npmcli/fs": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/@npmcli/fs/-/fs-1.1.1.tgz",
+      "integrity": "sha512-8KG5RD0GVP4ydEzRn/I4BNDuxDtqVbOdm8675T49OIG/NGhaK0pjPX7ZcDlvKYbA+ulvVK3ztfcF4uBdOxuJbQ==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "@gar/promisify": "^1.0.1",
+        "semver": "^7.3.5"
+      }
+    },
+    "node_modules/@npmcli/fs/node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "license": "ISC",
+      "optional": true,
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/@npmcli/move-file": {
+      "version": "1.1.2",
+      "resolved": "https://registry.npmjs.org/@npmcli/move-file/-/move-file-1.1.2.tgz",
+      "integrity": "sha512-1SUf/Cg2GzGDyaf15aR9St9TWlb+XvbZXWpDx8YKs7MLzMH/BCeopv+y9vzrzgkfykCGuWOlSu3mZhj2+FQcrg==",
+      "deprecated": "This functionality has been moved to @npmcli/fs",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "mkdirp": "^1.0.4",
+        "rimraf": "^3.0.2"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/@npmcli/move-file/node_modules/mkdirp": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/mkdirp/-/mkdirp-1.0.4.tgz",
+      "integrity": "sha512-vVqVZQyf3WLx2Shd0qJ9xuvqgAyKPLAiqITEtqW0oIUjzo3PePDd6fW9iFz30ef7Ysp/oiWqbhszeGWW2T6Gzw==",
+      "license": "MIT",
+      "optional": true,
+      "bin": {
+        "mkdirp": "bin/cmd.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
     "node_modules/@react-native-community/cli": {
       "version": "15.0.1",
       "resolved": "https://registry.npmjs.org/@react-native-community/cli/-/cli-15.0.1.tgz",
@@ -3296,6 +3357,16 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/@tootallnate/once": {
+      "version": "1.1.2",
+      "resolved": "https://registry.npmjs.org/@tootallnate/once/-/once-1.1.2.tgz",
+      "integrity": "sha512-RbzJvlNzmRq5c3O09UipeuXno4tA1FE6ikOjxZK0tuxVv3412l64l5t1W5pj4+rJq9vpkm/kwiR07aZXnsKPxw==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">= 6"
+      }
+    },
     "node_modules/@types/babel__core": {
       "version": "7.20.5",
       "resolved": "https://registry.npmjs.org/@types/babel__core/-/babel__core-7.20.5.tgz",
@@ -3483,6 +3554,20 @@
       "integrity": "sha512-I4q9QU9MQv4oEOz4tAHJtNz1cwuLxn2F3xcc2iV5WdqLPpUnj30aUuxt1mAxYTG+oe8CZMV/+6rU4S4gRDzqtQ==",
       "license": "MIT"
     },
+    "node_modules/@yarnpkg/lockfile": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/@yarnpkg/lockfile/-/lockfile-1.1.0.tgz",
+      "integrity": "sha512-GpSwvyXOcOOlV70vbnzjj4fW5xW/FdUF6nQEt1ENy7m4ZCczi1+/buVUPAqmGfqznsORNFzUMjctTIp8a9tuCQ==",
+      "dev": true,
+      "license": "BSD-2-Clause"
+    },
+    "node_modules/abbrev": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/abbrev/-/abbrev-1.1.1.tgz",
+      "integrity": "sha512-nne9/IiQ/hzIhY6pdDnbBtz7DjPTKrY00P/zvPSm5pOFkl6xuGrGnXn/VtTNNfNtAfZ9/1RtehkszU9qcTii0Q==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/abort-controller": {
       "version": "3.0.0",
       "resolved": "https://registry.npmjs.org/abort-controller/-/abort-controller-3.0.0.tgz",
@@ -3529,6 +3614,46 @@
         "node": ">=0.4.0"
       }
     },
+    "node_modules/agent-base": {
+      "version": "6.0.2",
+      "resolved": "https://registry.npmjs.org/agent-base/-/agent-base-6.0.2.tgz",
+      "integrity": "sha512-RZNwNclF7+MS/8bDg70amg32dyeZGZxiDuQmZxKLAlQjr3jGyLx+4Kkk58UO7D2QdgFIQCovuSuZESne6RG6XQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "debug": "4"
+      },
+      "engines": {
+        "node": ">= 6.0.0"
+      }
+    },
+    "node_modules/agentkeepalive": {
+      "version": "4.6.0",
+      "resolved": "https://registry.npmjs.org/agentkeepalive/-/agentkeepalive-4.6.0.tgz",
+      "integrity": "sha512-kja8j7PjmncONqaTsB8fQ+wE2mSU2DJ9D4XKoJ5PFWIdRMa6SLSN1ff4mOr4jCbfRSsxR4keIiySJU0N9T5hIQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "humanize-ms": "^1.2.1"
+      },
+      "engines": {
+        "node": ">= 8.0.0"
+      }
+    },
+    "node_modules/aggregate-error": {
+      "version": "3.1.0",
+      "resolved": "https://registry.npmjs.org/aggregate-error/-/aggregate-error-3.1.0.tgz",
+      "integrity": "sha512-4I7Td01quW/RpocfNayFdFVk1qSuoh0E7JrbRJ16nH01HhKFQ88INq9Sd+nd72zqRySlr9BmDA8xlEJ6vJMrYA==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "clean-stack": "^2.0.0",
+        "indent-string": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/anser": {
       "version": "1.4.10",
       "resolved": "https://registry.npmjs.org/anser/-/anser-1.4.10.tgz",
@@ -3617,6 +3742,28 @@
       ],
       "license": "MIT"
     },
+    "node_modules/aproba": {
+      "version": "2.1.0",
+      "resolved": "https://registry.npmjs.org/aproba/-/aproba-2.1.0.tgz",
+      "integrity": "sha512-tLIEcj5GuR2RSTnxNKdkK0dJ/GrC7P38sUkiDmDuHfsHmbagTFAxDVIBltoklXEVIQ/f14IL8IMJ5pn9Hez1Ew==",
+      "license": "ISC",
+      "optional": true
+    },
+    "node_modules/are-we-there-yet": {
+      "version": "3.0.1",
+      "resolved": "https://registry.npmjs.org/are-we-there-yet/-/are-we-there-yet-3.0.1.tgz",
+      "integrity": "sha512-QZW4EDmGwlYur0Yyf/b2uGucHQMa8aFUP7eu9ddR73vvhFyt4V0Vl3QHPcTNJ8l6qYOBdxgXdnBXQrHilfRQBg==",
+      "deprecated": "This package is no longer supported.",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "delegates": "^1.0.0",
+        "readable-stream": "^3.6.0"
+      },
+      "engines": {
+        "node": "^12.13.0 || ^14.15.0 || >=16.0.0"
+      }
+    },
     "node_modules/argparse": {
       "version": "1.0.10",
       "resolved": "https://registry.npmjs.org/argparse/-/argparse-1.0.10.tgz",
@@ -3626,6 +3773,12 @@
         "sprintf-js": "~1.0.2"
       }
     },
+    "node_modules/argsarray": {
+      "version": "0.0.1",
+      "resolved": "https://registry.npmjs.org/argsarray/-/argsarray-0.0.1.tgz",
+      "integrity": "sha512-u96dg2GcAKtpTrBdDoFIM7PjcBA+6rSP0OR94MOReNRyUECL6MtQt5XXmRr4qrftYaef9+l5hcpO5te7sML1Cg==",
+      "license": "WTFPL"
+    },
     "node_modules/asap": {
       "version": "2.0.6",
       "resolved": "https://registry.npmjs.org/asap/-/asap-2.0.6.tgz",
@@ -3660,6 +3813,16 @@
       "integrity": "sha512-csOlWGAcRFJaI6m+F2WKdnMKr4HhdhFVBk0H/QbJFMCr+uO2kwohwXQPxw/9OCxp05r5ghVBFSyioixx3gfkNQ==",
       "license": "MIT"
     },
+    "node_modules/at-least-node": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/at-least-node/-/at-least-node-1.0.0.tgz",
+      "integrity": "sha512-+q/t7Ekv1EDY2l6Gda6LLiX14rU9TV20Wa3ofeQmwPFZbOMo9DXrLbOjFaaclkXKWidIaopwAObQDqwWtGUjqg==",
+      "dev": true,
+      "license": "ISC",
+      "engines": {
+        "node": ">= 4.0.0"
+      }
+    },
     "node_modules/babel-core": {
       "version": "7.0.0-bridge.0",
       "resolved": "https://registry.npmjs.org/babel-core/-/babel-core-7.0.0-bridge.0.tgz",
@@ -3873,6 +4036,16 @@
         "node": ">=6.0.0"
       }
     },
+    "node_modules/bindings": {
+      "version": "1.5.0",
+      "resolved": "https://registry.npmjs.org/bindings/-/bindings-1.5.0.tgz",
+      "integrity": "sha512-p2q/t/mhvuOj/UeLlV6566GD/guowlr0hHxClI0W9m7MWYkL1F0hLo+0Aexs9HSPCtR1SXQ0TD3MMKrXZajbiQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "file-uri-to-path": "1.0.0"
+      }
+    },
     "node_modules/bl": {
       "version": "4.1.0",
       "resolved": "https://registry.npmjs.org/bl/-/bl-4.1.0.tgz",
@@ -3990,6 +4163,119 @@
         "node": ">= 0.8"
       }
     },
+    "node_modules/cacache": {
+      "version": "15.3.0",
+      "resolved": "https://registry.npmjs.org/cacache/-/cacache-15.3.0.tgz",
+      "integrity": "sha512-VVdYzXEn+cnbXpFgWs5hTT7OScegHVmLhJIR8Ufqk3iFD6A6j5iSX1KuBTfNEv4tdJWE2PzA6IVFtcLC7fN9wQ==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "@npmcli/fs": "^1.0.0",
+        "@npmcli/move-file": "^1.0.1",
+        "chownr": "^2.0.0",
+        "fs-minipass": "^2.0.0",
+        "glob": "^7.1.4",
+        "infer-owner": "^1.0.4",
+        "lru-cache": "^6.0.0",
+        "minipass": "^3.1.1",
+        "minipass-collect": "^1.0.2",
+        "minipass-flush": "^1.0.5",
+        "minipass-pipeline": "^1.2.2",
+        "mkdirp": "^1.0.3",
+        "p-map": "^4.0.0",
+        "promise-inflight": "^1.0.1",
+        "rimraf": "^3.0.2",
+        "ssri": "^8.0.1",
+        "tar": "^6.0.2",
+        "unique-filename": "^1.1.1"
+      },
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/cacache/node_modules/lru-cache": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-6.0.0.tgz",
+      "integrity": "sha512-Jo6dJ04CmSjuznwJSS3pUeWmd/H0ffTlkXXgwZi+eq1UCmqQwCh+eLsYOYCwY991i2Fah4h1BEMCx4qThGbsiA==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "yallist": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/cacache/node_modules/mkdirp": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/mkdirp/-/mkdirp-1.0.4.tgz",
+      "integrity": "sha512-vVqVZQyf3WLx2Shd0qJ9xuvqgAyKPLAiqITEtqW0oIUjzo3PePDd6fW9iFz30ef7Ysp/oiWqbhszeGWW2T6Gzw==",
+      "license": "MIT",
+      "optional": true,
+      "bin": {
+        "mkdirp": "bin/cmd.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/cacache/node_modules/yallist": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/yallist/-/yallist-4.0.0.tgz",
+      "integrity": "sha512-3wdGidZyq5PB084XLES5TpOSRA3wjXAlIWMhum2kRcv/41Sn2emQ0dycQW4uZXLejwKvg6EsvbdlVL+FYEct7A==",
+      "license": "ISC",
+      "optional": true
+    },
+    "node_modules/call-bind": {
+      "version": "1.0.9",
+      "resolved": "https://registry.npmjs.org/call-bind/-/call-bind-1.0.9.tgz",
+      "integrity": "sha512-a/hy+pNsFUTR+Iz8TCJvXudKVLAnz/DyeSUo10I5yvFDQJBFU2s9uqQpoSrJlroHUKoKqzg+epxyP9lqFdzfBQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind-apply-helpers": "^1.0.2",
+        "es-define-property": "^1.0.1",
+        "get-intrinsic": "^1.3.0",
+        "set-function-length": "^1.2.2"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/call-bind-apply-helpers": {
+      "version": "1.0.2",
+      "resolved": "https://registry.npmjs.org/call-bind-apply-helpers/-/call-bind-apply-helpers-1.0.2.tgz",
+      "integrity": "sha512-Sp1ablJ0ivDkSzjcaJdxEunN5/XvksFJ2sMBFfq6x0ryhQV/2b/KwFe21cMpmHtPOSij8K99/wSfoEuTObmuMQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "es-errors": "^1.3.0",
+        "function-bind": "^1.1.2"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
+    "node_modules/call-bound": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/call-bound/-/call-bound-1.0.4.tgz",
+      "integrity": "sha512-+ys997U96po4Kx/ABpBCqhA9EuxJaQWDQg7295H4hBphv3IZg0boBKuwYpt4YXp6MZ5AmZQnU/tyMTlRpaSejg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind-apply-helpers": "^1.0.2",
+        "get-intrinsic": "^1.3.0"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
     "node_modules/caller-callsite": {
       "version": "2.0.0",
       "resolved": "https://registry.npmjs.org/caller-callsite/-/caller-callsite-2.0.0.tgz",
@@ -4088,6 +4374,16 @@
         "node": ">=10"
       }
     },
+    "node_modules/chownr": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/chownr/-/chownr-2.0.0.tgz",
+      "integrity": "sha512-bIomtDF5KGpdogkLd9VspvFzk9KfpyyGlS8YFVZl7TGPBHL5snIOnxeshwVgPteQ9b4Eydl+pVbIyE1DcvCWgQ==",
+      "license": "ISC",
+      "optional": true,
+      "engines": {
+        "node": ">=10"
+      }
+    },
     "node_modules/chrome-launcher": {
       "version": "0.15.2",
       "resolved": "https://registry.npmjs.org/chrome-launcher/-/chrome-launcher-0.15.2.tgz",
@@ -4178,6 +4474,16 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/clean-stack": {
+      "version": "2.2.0",
+      "resolved": "https://registry.npmjs.org/clean-stack/-/clean-stack-2.2.0.tgz",
+      "integrity": "sha512-4diC9HaTE+KRAMWhDhrGOECgWZxoevMc5TlkObMqNSsVU62PYzXZ/SMTjzyGAFF1YusgxGcSWTEXBhp0CPwQ1A==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">=6"
+      }
+    },
     "node_modules/cli-cursor": {
       "version": "3.1.0",
       "resolved": "https://registry.npmjs.org/cli-cursor/-/cli-cursor-3.1.0.tgz",
@@ -4290,6 +4596,16 @@
       "integrity": "sha512-dOy+3AuW3a2wNbZHIuMZpTcgjGuLU/uBL/ubcZF9OXbDo8ff4O8yVp5Bf0efS8uEoYo5q4Fx7dY9OgQGXgAsQA==",
       "license": "MIT"
     },
+    "node_modules/color-support": {
+      "version": "1.1.3",
+      "resolved": "https://registry.npmjs.org/color-support/-/color-support-1.1.3.tgz",
+      "integrity": "sha512-qiBjkpbMLO/HL68y+lh4q0/O1MZFj2RX6X/KmMa3+gJD3z+WwI1ZzDHysvqHGS3mP6mznPckpXmw1nI9cJjyRg==",
+      "license": "ISC",
+      "optional": true,
+      "bin": {
+        "color-support": "bin.js"
+      }
+    },
     "node_modules/colorette": {
       "version": "1.4.0",
       "resolved": "https://registry.npmjs.org/colorette/-/colorette-1.4.0.tgz",
@@ -4405,6 +4721,13 @@
       "integrity": "sha512-Tpp60P6IUJDTuOq/5Z8cdskzJujfwqfOTkrwIwj7IRISpnkJnT6SyJ4PCPnGMoFjC9ddhal5KVIYtAt97ix05A==",
       "license": "MIT"
     },
+    "node_modules/console-control-strings": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/console-control-strings/-/console-control-strings-1.1.0.tgz",
+      "integrity": "sha512-ty/fTekppD2fIwRvnZAVdeOiGd1c7YXEixbgJTNzqcxJWKQnjJ/V1bNEEE6hygpM3WjwHFUVK6HTjWSzV4a8sQ==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/convert-source-map": {
       "version": "2.0.0",
       "resolved": "https://registry.npmjs.org/convert-source-map/-/convert-source-map-2.0.0.tgz",
@@ -4561,6 +4884,22 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/decompress-response": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/decompress-response/-/decompress-response-6.0.0.tgz",
+      "integrity": "sha512-aW35yZM6Bb/4oJlZncMH2LCoZtJXTRxES17vE3hoRiowU2kWHaJKFkSBDnDR+cm9J+9QhXmREyIfv0pji9ejCQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "mimic-response": "^3.1.0"
+      },
+      "engines": {
+        "node": ">=10"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/sindresorhus"
+      }
+    },
     "node_modules/dedent": {
       "version": "1.7.2",
       "resolved": "https://registry.npmjs.org/dedent/-/dedent-1.7.2.tgz",
@@ -4576,6 +4915,16 @@
         }
       }
     },
+    "node_modules/deep-extend": {
+      "version": "0.6.0",
+      "resolved": "https://registry.npmjs.org/deep-extend/-/deep-extend-0.6.0.tgz",
+      "integrity": "sha512-LOHxIOaPYdHlJRtCQfDIVZtfw/ufM8+rVj649RIHzcm/vGwQRXFt6OPqIFWsm2XEMrNIEtWR64sY1LEKD2vAOA==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">=4.0.0"
+      }
+    },
     "node_modules/deepmerge": {
       "version": "4.3.1",
       "resolved": "https://registry.npmjs.org/deepmerge/-/deepmerge-4.3.1.tgz",
@@ -4599,6 +4948,31 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/define-data-property": {
+      "version": "1.1.4",
+      "resolved": "https://registry.npmjs.org/define-data-property/-/define-data-property-1.1.4.tgz",
+      "integrity": "sha512-rBMvIzlpA8v6E+SJZoo++HAYqsLrkg7MSfIinMPFhmkorw7X+dOXVJQs+QT69zGkzMyfDnIMN2Wid1+NbL3T+A==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "es-define-property": "^1.0.0",
+        "es-errors": "^1.3.0",
+        "gopd": "^1.0.1"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/delegates": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/delegates/-/delegates-1.0.0.tgz",
+      "integrity": "sha512-bd2L678uiWATM6m5Z1VzNCErI3jiGzt6HGY8OVICs40JQq/HALfbyNJmp0UDakEY4pMMaN0Ly5om/B1VI/+xfQ==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/depd": {
       "version": "2.0.0",
       "resolved": "https://registry.npmjs.org/depd/-/depd-2.0.0.tgz",
@@ -4618,6 +4992,16 @@
         "npm": "1.2.8000 || >= 1.4.16"
       }
     },
+    "node_modules/detect-libc": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/detect-libc/-/detect-libc-2.1.2.tgz",
+      "integrity": "sha512-Btj2BOOO83o3WyH59e8MgXsxEQVcarkUOpEYrubB0urwnN10yQ364rsiByU11nZlqWYZm05i/of7io4mzihBtQ==",
+      "license": "Apache-2.0",
+      "optional": true,
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/detect-newline": {
       "version": "3.1.0",
       "resolved": "https://registry.npmjs.org/detect-newline/-/detect-newline-3.1.0.tgz",
@@ -4638,6 +5022,21 @@
         "node": "^14.15.0 || ^16.10.0 || >=18.0.0"
       }
     },
+    "node_modules/dunder-proto": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/dunder-proto/-/dunder-proto-1.0.1.tgz",
+      "integrity": "sha512-KIN/nDJBQRcXw0MLVhZE9iQHmG68qAVIBg9CqmUYjmQIhgij9U5MFvrqkUL5FbtyyzZuOeOt0zdeRe4UY7ct+A==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind-apply-helpers": "^1.0.1",
+        "es-errors": "^1.3.0",
+        "gopd": "^1.2.0"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/ee-first": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/ee-first/-/ee-first-1.1.1.tgz",
@@ -4678,13 +5077,33 @@
         "node": ">= 0.8"
       }
     },
-    "node_modules/env-paths": {
-      "version": "2.2.1",
-      "resolved": "https://registry.npmjs.org/env-paths/-/env-paths-2.2.1.tgz",
-      "integrity": "sha512-+h1lkLKhZMTYjog1VEpJNG7NZJWcuc2DDk/qsqSTRRCOXiLjeQ1d1/udrUGhqMxUgAlwKNZ0cf2uqan5GLuS2A==",
-      "devOptional": true,
+    "node_modules/encoding": {
+      "version": "0.1.13",
+      "resolved": "https://registry.npmjs.org/encoding/-/encoding-0.1.13.tgz",
+      "integrity": "sha512-ETBauow1T35Y/WZMkio9jiM0Z5xjHHmJ4XmjZOq1l/dXz3lr2sRn87nJy20RupqSh1F2m3HHPSp8ShIPQJrJ3A==",
       "license": "MIT",
-      "engines": {
+      "optional": true,
+      "dependencies": {
+        "iconv-lite": "^0.6.2"
+      }
+    },
+    "node_modules/end-of-stream": {
+      "version": "1.4.5",
+      "resolved": "https://registry.npmjs.org/end-of-stream/-/end-of-stream-1.4.5.tgz",
+      "integrity": "sha512-ooEGc6HP26xXq/N+GCGOT0JKCLDGrq2bQUZrQ7gyrJiZANJ/8YDTxTpQBXGMn+WbIQXNVpyWymm7KYVICQnyOg==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "once": "^1.4.0"
+      }
+    },
+    "node_modules/env-paths": {
+      "version": "2.2.1",
+      "resolved": "https://registry.npmjs.org/env-paths/-/env-paths-2.2.1.tgz",
+      "integrity": "sha512-+h1lkLKhZMTYjog1VEpJNG7NZJWcuc2DDk/qsqSTRRCOXiLjeQ1d1/udrUGhqMxUgAlwKNZ0cf2uqan5GLuS2A==",
+      "devOptional": true,
+      "license": "MIT",
+      "engines": {
         "node": ">=6"
       }
     },
@@ -4701,6 +5120,13 @@
         "node": ">=4"
       }
     },
+    "node_modules/err-code": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/err-code/-/err-code-2.0.3.tgz",
+      "integrity": "sha512-2bmlRpNKBxT/CRmPOlyISQpNj+qSeYvcym/uT0Jx2bMOlKLtSy1ZmLuVxSEKKyor/N5yhvp/ZiG1oE3DEYMSFA==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/error-ex": {
       "version": "1.3.4",
       "resolved": "https://registry.npmjs.org/error-ex/-/error-ex-1.3.4.tgz",
@@ -4737,6 +5163,16 @@
         "url": "https://opencollective.com/express"
       }
     },
+    "node_modules/es-define-property": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/es-define-property/-/es-define-property-1.0.1.tgz",
+      "integrity": "sha512-e3nRfgfUZ4rNGL232gUgX06QNyyez04KdjFrF+LTRoOXmrOgFKDg4BCdsjW8EnT69eqdYGmRpJwiPVYNrCaW3g==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/es-errors": {
       "version": "1.3.0",
       "resolved": "https://registry.npmjs.org/es-errors/-/es-errors-1.3.0.tgz",
@@ -4746,6 +5182,19 @@
         "node": ">= 0.4"
       }
     },
+    "node_modules/es-object-atoms": {
+      "version": "1.1.2",
+      "resolved": "https://registry.npmjs.org/es-object-atoms/-/es-object-atoms-1.1.2.tgz",
+      "integrity": "sha512-HWcBoN6NileqtSydK2FqHbS/LoDd2pqrnQHLyJzBj4kOp/ky2MWMN694xOfkK8/SnUsW2DH7EfyVlydKCsm1Zw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "es-errors": "^1.3.0"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/escalade": {
       "version": "3.2.0",
       "resolved": "https://registry.npmjs.org/escalade/-/escalade-3.2.0.tgz",
@@ -4846,6 +5295,16 @@
         "node": ">= 0.8.0"
       }
     },
+    "node_modules/expand-template": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/expand-template/-/expand-template-2.0.3.tgz",
+      "integrity": "sha512-XYfuKMvj4O35f/pOXLObndIRvyQ+/+6AhODh+OKWj9S9498pHHn/IMszH+gt0fBCRWMNfk1ZSp5x3AifmnI2vg==",
+      "license": "(MIT OR WTFPL)",
+      "optional": true,
+      "engines": {
+        "node": ">=6"
+      }
+    },
     "node_modules/expect": {
       "version": "29.7.0",
       "resolved": "https://registry.npmjs.org/expect/-/expect-29.7.0.tgz",
@@ -4930,6 +5389,13 @@
         "bser": "2.1.1"
       }
     },
+    "node_modules/file-uri-to-path": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/file-uri-to-path/-/file-uri-to-path-1.0.0.tgz",
+      "integrity": "sha512-0Zt+s3L7Vf1biwWZ29aARiVYLx7iMGnEUl9x33fbB/j3jR81u/O2LbqK+Bm1CDSNDKVtJ/YjwY7TUd5SkeLQLw==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/fill-range": {
       "version": "7.1.1",
       "resolved": "https://registry.npmjs.org/fill-range/-/fill-range-7.1.1.tgz",
@@ -5101,6 +5567,16 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/find-yarn-workspace-root": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/find-yarn-workspace-root/-/find-yarn-workspace-root-2.0.0.tgz",
+      "integrity": "sha512-1IMnbjt4KzsQfnhnzNd8wUEgXZ44IzZaZmnLYx7D5FZlaHt2gW20Cri8Q+E/t5tIj4+epTBub+2Zxu/vNILzqQ==",
+      "dev": true,
+      "license": "Apache-2.0",
+      "dependencies": {
+        "micromatch": "^4.0.2"
+      }
+    },
     "node_modules/flow-enums-runtime": {
       "version": "0.0.6",
       "resolved": "https://registry.npmjs.org/flow-enums-runtime/-/flow-enums-runtime-0.0.6.tgz",
@@ -5137,6 +5613,13 @@
         "node": ">= 0.6"
       }
     },
+    "node_modules/fs-constants": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/fs-constants/-/fs-constants-1.0.0.tgz",
+      "integrity": "sha512-y6OAwoSIf7FyjMIv94u+b5rdheZEjzR63GTyZJm5qh4Bi+2YgwLCcI/fPFZkL5PSixOt6ZNKm+w+Hfp/Bciwow==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/fs-extra": {
       "version": "8.1.0",
       "resolved": "https://registry.npmjs.org/fs-extra/-/fs-extra-8.1.0.tgz",
@@ -5152,6 +5635,19 @@
         "node": ">=6 <7 || >=8"
       }
     },
+    "node_modules/fs-minipass": {
+      "version": "2.1.0",
+      "resolved": "https://registry.npmjs.org/fs-minipass/-/fs-minipass-2.1.0.tgz",
+      "integrity": "sha512-V/JgOLFCS+R6Vcq0slCuaeWEdNC3ouDlJMNIsacH2VtALiu9mV4LPrHc5cDl8k5aw6J8jwgWWpiTo5RYhmIzvg==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0"
+      },
+      "engines": {
+        "node": ">= 8"
+      }
+    },
     "node_modules/fs.realpath": {
       "version": "1.0.0",
       "resolved": "https://registry.npmjs.org/fs.realpath/-/fs.realpath-1.0.0.tgz",
@@ -5181,6 +5677,40 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/gauge": {
+      "version": "4.0.4",
+      "resolved": "https://registry.npmjs.org/gauge/-/gauge-4.0.4.tgz",
+      "integrity": "sha512-f9m+BEN5jkg6a0fZjleidjN51VE1X+mPFQ2DJ0uv1V39oCLCbsGe6yjbBnp7eK7z/+GAon99a3nHuqbuuthyPg==",
+      "deprecated": "This package is no longer supported.",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "aproba": "^1.0.3 || ^2.0.0",
+        "color-support": "^1.1.3",
+        "console-control-strings": "^1.1.0",
+        "has-unicode": "^2.0.1",
+        "signal-exit": "^3.0.7",
+        "string-width": "^4.2.3",
+        "strip-ansi": "^6.0.1",
+        "wide-align": "^1.1.5"
+      },
+      "engines": {
+        "node": "^12.13.0 || ^14.15.0 || >=16.0.0"
+      }
+    },
+    "node_modules/gauge/node_modules/strip-ansi": {
+      "version": "6.0.1",
+      "resolved": "https://registry.npmjs.org/strip-ansi/-/strip-ansi-6.0.1.tgz",
+      "integrity": "sha512-Y38VPSHcqkFrCpFnQ9vuSXmquuv5oXOKpGeT6aGrr3o3Gc9AlVa6JBfUSOCnbxGGZF+/0ooI7KrPuUSztUdU5A==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "ansi-regex": "^5.0.1"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
     "node_modules/gensync": {
       "version": "1.0.0-beta.2",
       "resolved": "https://registry.npmjs.org/gensync/-/gensync-1.0.0-beta.2.tgz",
@@ -5199,6 +5729,31 @@
         "node": "6.* || 8.* || >= 10.*"
       }
     },
+    "node_modules/get-intrinsic": {
+      "version": "1.3.0",
+      "resolved": "https://registry.npmjs.org/get-intrinsic/-/get-intrinsic-1.3.0.tgz",
+      "integrity": "sha512-9fSjSaos/fRIVIp+xSJlE6lfwhES7LNtKaCBIamHsjr2na1BiABJPo0mOjjz8GJDURarmCPGqaiVg5mfjb98CQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind-apply-helpers": "^1.0.2",
+        "es-define-property": "^1.0.1",
+        "es-errors": "^1.3.0",
+        "es-object-atoms": "^1.1.1",
+        "function-bind": "^1.1.2",
+        "get-proto": "^1.0.1",
+        "gopd": "^1.2.0",
+        "has-symbols": "^1.1.0",
+        "hasown": "^2.0.2",
+        "math-intrinsics": "^1.1.0"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
     "node_modules/get-package-type": {
       "version": "0.1.0",
       "resolved": "https://registry.npmjs.org/get-package-type/-/get-package-type-0.1.0.tgz",
@@ -5208,6 +5763,20 @@
         "node": ">=8.0.0"
       }
     },
+    "node_modules/get-proto": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/get-proto/-/get-proto-1.0.1.tgz",
+      "integrity": "sha512-sTSfBjoXBp89JvIKIefqw7U2CCebsc74kiY6awiGogKtoSGbgjYE/G/+l9sF3MWFPNc9IcoOC4ODfKHfxFmp0g==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "dunder-proto": "^1.0.1",
+        "es-object-atoms": "^1.0.0"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/get-stream": {
       "version": "6.0.1",
       "resolved": "https://registry.npmjs.org/get-stream/-/get-stream-6.0.1.tgz",
@@ -5220,6 +5789,13 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/github-from-package": {
+      "version": "0.0.0",
+      "resolved": "https://registry.npmjs.org/github-from-package/-/github-from-package-0.0.0.tgz",
+      "integrity": "sha512-SyHy3T1v2NUXn29OsWdxmK6RwHD+vkj3v8en8AOBZ1wBQ/hCAQ5bAQTD02kW4W9tUp/3Qh6J8r9EvntiyCmOOw==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/glob": {
       "version": "7.2.3",
       "resolved": "https://registry.npmjs.org/glob/-/glob-7.2.3.tgz",
@@ -5254,6 +5830,19 @@
         "node": ">= 6"
       }
     },
+    "node_modules/gopd": {
+      "version": "1.2.0",
+      "resolved": "https://registry.npmjs.org/gopd/-/gopd-1.2.0.tgz",
+      "integrity": "sha512-ZUKRh6/kUFoAiTAtTYPZJ3hw9wNxx+BIBOijnlG9PnrJsCcSjs1wyyD6vJpaYtgnzDrKYRSqf3OO6Rfa93xsRg==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
     "node_modules/graceful-fs": {
       "version": "4.2.11",
       "resolved": "https://registry.npmjs.org/graceful-fs/-/graceful-fs-4.2.11.tgz",
@@ -5269,6 +5858,39 @@
         "node": ">=8"
       }
     },
+    "node_modules/has-property-descriptors": {
+      "version": "1.0.2",
+      "resolved": "https://registry.npmjs.org/has-property-descriptors/-/has-property-descriptors-1.0.2.tgz",
+      "integrity": "sha512-55JNKuIW+vq4Ke1BjOTjM2YctQIvCT7GFzHwmfZPGo5wnrgkid0YQtnAleFSqumZm4az3n2BS+erby5ipJdgrg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "es-define-property": "^1.0.0"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/has-symbols": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/has-symbols/-/has-symbols-1.1.0.tgz",
+      "integrity": "sha512-1cDNdwJ2Jaohmb3sg4OmKaMBwuC48sYni5HUw2DvsC8LjGTLK9h+eb1X6RyuOHe4hT0ULCW68iomhjUoKUqlPQ==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
+    "node_modules/has-unicode": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/has-unicode/-/has-unicode-2.0.1.tgz",
+      "integrity": "sha512-8Rf9Y83NBReMnx0gFzA8JImQACstCYWUplepDa9xprwwtmgEZUF0h/i5xSA625zB/I37EtrswSST6OXxwaaIJQ==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/hasown": {
       "version": "2.0.4",
       "resolved": "https://registry.npmjs.org/hasown/-/hasown-2.0.4.tgz",
@@ -5303,6 +5925,13 @@
       "dev": true,
       "license": "MIT"
     },
+    "node_modules/http-cache-semantics": {
+      "version": "4.2.0",
+      "resolved": "https://registry.npmjs.org/http-cache-semantics/-/http-cache-semantics-4.2.0.tgz",
+      "integrity": "sha512-dTxcvPXqPvXBQpq5dUr6mEMJX4oIEFv6bwom3FDwKRDsuIjjJGANqhBuoAn9c1RQJIdAKav33ED65E2ys+87QQ==",
+      "license": "BSD-2-Clause",
+      "optional": true
+    },
     "node_modules/http-errors": {
       "version": "2.0.1",
       "resolved": "https://registry.npmjs.org/http-errors/-/http-errors-2.0.1.tgz",
@@ -5332,6 +5961,35 @@
         "node": ">= 0.8"
       }
     },
+    "node_modules/http-proxy-agent": {
+      "version": "4.0.1",
+      "resolved": "https://registry.npmjs.org/http-proxy-agent/-/http-proxy-agent-4.0.1.tgz",
+      "integrity": "sha512-k0zdNgqWTGA6aeIRVpvfVob4fL52dTfaehylg0Y4UvSySvOq/Y+BOyPrgpUrA7HylqvU8vIZGsRuXmspskV0Tg==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "@tootallnate/once": "1",
+        "agent-base": "6",
+        "debug": "4"
+      },
+      "engines": {
+        "node": ">= 6"
+      }
+    },
+    "node_modules/https-proxy-agent": {
+      "version": "5.0.1",
+      "resolved": "https://registry.npmjs.org/https-proxy-agent/-/https-proxy-agent-5.0.1.tgz",
+      "integrity": "sha512-dFcAjpTQFgoLMzC2VwU+C/CbS7uRL0lWmxDITmqm7C+7F0Odmj6s9l6alZc6AELXhrnggM2CeWSXHGOdX2YtwA==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "agent-base": "6",
+        "debug": "4"
+      },
+      "engines": {
+        "node": ">= 6"
+      }
+    },
     "node_modules/human-signals": {
       "version": "2.1.0",
       "resolved": "https://registry.npmjs.org/human-signals/-/human-signals-2.1.0.tgz",
@@ -5341,6 +5999,29 @@
         "node": ">=10.17.0"
       }
     },
+    "node_modules/humanize-ms": {
+      "version": "1.2.1",
+      "resolved": "https://registry.npmjs.org/humanize-ms/-/humanize-ms-1.2.1.tgz",
+      "integrity": "sha512-Fl70vYtsAFb/C06PTS9dZBo7ihau+Tu/DNCk/OyHhea07S+aeMWpFFkUaXRa8fI+ScZbEI8dfSxwY7gxZ9SAVQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "ms": "^2.0.0"
+      }
+    },
+    "node_modules/iconv-lite": {
+      "version": "0.6.3",
+      "resolved": "https://registry.npmjs.org/iconv-lite/-/iconv-lite-0.6.3.tgz",
+      "integrity": "sha512-4fCk79wshMdzMp2rH06qWrJE4iolqLhCUH+OiuIgU++RB0+94NlDL81atO7GX55uUKueo0txHNtvEyI6D7WdMw==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "safer-buffer": ">= 2.1.2 < 3.0.0"
+      },
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/ieee754": {
       "version": "1.2.1",
       "resolved": "https://registry.npmjs.org/ieee754/-/ieee754-1.2.1.tgz",
@@ -5377,6 +6058,12 @@
         "node": ">=16.x"
       }
     },
+    "node_modules/immediate": {
+      "version": "3.3.0",
+      "resolved": "https://registry.npmjs.org/immediate/-/immediate-3.3.0.tgz",
+      "integrity": "sha512-HR7EVodfFUdQCTIeySw+WDRFJlPcLOJbXfwwZ7Oom6tjsvZ3bOkCDJHehQC3nxJrv7+f9XecwazynjU8e4Vw3Q==",
+      "license": "MIT"
+    },
     "node_modules/import-fresh": {
       "version": "3.3.1",
       "resolved": "https://registry.npmjs.org/import-fresh/-/import-fresh-3.3.1.tgz",
@@ -5437,12 +6124,19 @@
       "version": "4.0.0",
       "resolved": "https://registry.npmjs.org/indent-string/-/indent-string-4.0.0.tgz",
       "integrity": "sha512-EdDDZu4A2OyIK7Lr/2zG+w5jmbuk1DVBnEwREQvBzspBJkCEbRa8GxU1lghYcaGJCnRWibjDXlq779X1/y5xwg==",
-      "dev": true,
+      "devOptional": true,
       "license": "MIT",
       "engines": {
         "node": ">=8"
       }
     },
+    "node_modules/infer-owner": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/infer-owner/-/infer-owner-1.0.4.tgz",
+      "integrity": "sha512-IClj+Xz94+d7irH5qRyfJonOdfTzuDaifE6ZPWfx0N0+/ATZCbuTPq2prFl526urkQd90WyUKIh1DfBQ2hMz9A==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/inflight": {
       "version": "1.0.6",
       "resolved": "https://registry.npmjs.org/inflight/-/inflight-1.0.6.tgz",
@@ -5460,6 +6154,13 @@
       "integrity": "sha512-k/vGaX4/Yla3WzyMCvTQOXYeIHvqOKtnqBduzTHpzpQZzAskKMhZ2K+EnBiSM9zGSoIFeMpXKxa4dYeZIQqewQ==",
       "license": "ISC"
     },
+    "node_modules/ini": {
+      "version": "1.3.8",
+      "resolved": "https://registry.npmjs.org/ini/-/ini-1.3.8.tgz",
+      "integrity": "sha512-JV/yugV2uzW5iMRSiZAyDtQd+nxtUnjeLt0acNdw98kKLrvuRVyB80tsREOE7yvGVgalhZ6RNXCmEHkUKBKxew==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/invariant": {
       "version": "2.2.4",
       "resolved": "https://registry.npmjs.org/invariant/-/invariant-2.2.4.tgz",
@@ -5469,6 +6170,16 @@
         "loose-envify": "^1.0.0"
       }
     },
+    "node_modules/ip-address": {
+      "version": "10.5.0",
+      "resolved": "https://registry.npmjs.org/ip-address/-/ip-address-10.5.0.tgz",
+      "integrity": "sha512-R5SnVLJmgYYvf2F2ZgwSBnelz5G4q5AxIC277GDfUaNbrZKNANcBC7RHqYYePlszf4kBolVkJauG0ZjHHFh55g==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">= 12"
+      }
+    },
     "node_modules/is-arrayish": {
       "version": "0.2.1",
       "resolved": "https://registry.npmjs.org/is-arrayish/-/is-arrayish-0.2.1.tgz",
@@ -5567,6 +6278,13 @@
         "node": ">=8"
       }
     },
+    "node_modules/is-lambda": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/is-lambda/-/is-lambda-1.0.1.tgz",
+      "integrity": "sha512-z7CMFGNrENq5iFB9Bqo64Xk6Y9sg+epq1myIcdHaGnbMTYOxvzsEtdYqQUylB7LxfkvgrrjP32T6Ywciio9UIQ==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/is-number": {
       "version": "7.0.0",
       "resolved": "https://registry.npmjs.org/is-number/-/is-number-7.0.0.tgz",
@@ -5623,6 +6341,13 @@
         "node": ">=4"
       }
     },
+    "node_modules/isarray": {
+      "version": "2.0.5",
+      "resolved": "https://registry.npmjs.org/isarray/-/isarray-2.0.5.tgz",
+      "integrity": "sha512-xHjhDr3cNBK0BzdUJSPXZntQUx/mwMS5Rw4A7lPJ90XGAO6ISP/ePDNuo0vhqOZU+UD5JoodwCAAoZQd3FeAKw==",
+      "dev": true,
+      "license": "MIT"
+    },
     "node_modules/isexe": {
       "version": "2.0.0",
       "resolved": "https://registry.npmjs.org/isexe/-/isexe-2.0.0.tgz",
@@ -6715,6 +7440,26 @@
       "devOptional": true,
       "license": "MIT"
     },
+    "node_modules/json-stable-stringify": {
+      "version": "1.3.0",
+      "resolved": "https://registry.npmjs.org/json-stable-stringify/-/json-stable-stringify-1.3.0.tgz",
+      "integrity": "sha512-qtYiSSFlwot9XHtF9bD9c7rwKjr+RecWT//ZnPvSmEjpV5mmPOCN4j8UjY5hbjNkOwZ/jQv3J6R1/pL7RwgMsg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "call-bind": "^1.0.8",
+        "call-bound": "^1.0.4",
+        "isarray": "^2.0.5",
+        "jsonify": "^0.0.1",
+        "object-keys": "^1.1.1"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
     "node_modules/json5": {
       "version": "2.2.3",
       "resolved": "https://registry.npmjs.org/json5/-/json5-2.2.3.tgz",
@@ -6737,6 +7482,16 @@
         "graceful-fs": "^4.1.6"
       }
     },
+    "node_modules/jsonify": {
+      "version": "0.0.1",
+      "resolved": "https://registry.npmjs.org/jsonify/-/jsonify-0.0.1.tgz",
+      "integrity": "sha512-2/Ki0GcmuqSrgFyelQq9M05y7PS0mEwuIzrf3f1fPqkVDVRvZrPZtVSMHxdgo8Aq0sxAOb/cr2aqqA3LeWHVPg==",
+      "dev": true,
+      "license": "Public Domain",
+      "funding": {
+        "url": "https://github.com/sponsors/ljharb"
+      }
+    },
     "node_modules/kind-of": {
       "version": "6.0.3",
       "resolved": "https://registry.npmjs.org/kind-of/-/kind-of-6.0.3.tgz",
@@ -6746,6 +7501,16 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/klaw-sync": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/klaw-sync/-/klaw-sync-6.0.0.tgz",
+      "integrity": "sha512-nIeuVSzdCCs6TDPTqI8w1Yre34sSq7AkZ4B3sfOBbI2CgVSB4Du4aLQijFU2+lhAFCwt9+42Hel6lQNIv6AntQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "graceful-fs": "^4.1.11"
+      }
+    },
     "node_modules/kleur": {
       "version": "3.0.3",
       "resolved": "https://registry.npmjs.org/kleur/-/kleur-3.0.3.tgz",
@@ -6819,12 +7584,24 @@
       "integrity": "sha512-FT1yDzDYEoYWhnSGnpE/4Kj1fLZkDFyqRb7fNt6FdYOSxlUWAtp42Eh6Wb0rGIv/m9Bgo7x4GhQbm5Ys4SG5ow==",
       "license": "MIT"
     },
+    "node_modules/lodash.map": {
+      "version": "4.6.0",
+      "resolved": "https://registry.npmjs.org/lodash.map/-/lodash.map-4.6.0.tgz",
+      "integrity": "sha512-worNHGKLDetmcEYDvh2stPCrrQRkP20E4l0iIS7F8EvzMqBBi7ltvFN5m1HvTf1P7Jk1txKhvFcmYsCr8O2F1Q==",
+      "license": "MIT"
+    },
     "node_modules/lodash.throttle": {
       "version": "4.1.1",
       "resolved": "https://registry.npmjs.org/lodash.throttle/-/lodash.throttle-4.1.1.tgz",
       "integrity": "sha512-wIkUCfVKpVsWo3JSZlc+8MB5it+2AN5W8J7YVMST30UrvcQNZ1Okbj+rbVniijTWE6FGYy4XJq/rHkas8qJMLQ==",
       "license": "MIT"
     },
+    "node_modules/lodash.zipobject": {
+      "version": "4.18.0",
+      "resolved": "https://registry.npmjs.org/lodash.zipobject/-/lodash.zipobject-4.18.0.tgz",
+      "integrity": "sha512-crXabtkOHmlyg/VoiR0wzEjbl8b+owY2Z8wBhoyoGPya/9YkUsxpCe7UJkL+ZtQFBBQXnk08kz9W0iieQNeu9w==",
+      "license": "MIT"
+    },
     "node_modules/log-symbols": {
       "version": "4.1.0",
       "resolved": "https://registry.npmjs.org/log-symbols/-/log-symbols-4.1.0.tgz",
@@ -7047,6 +7824,54 @@
         "node": ">=10"
       }
     },
+    "node_modules/make-fetch-happen": {
+      "version": "9.1.0",
+      "resolved": "https://registry.npmjs.org/make-fetch-happen/-/make-fetch-happen-9.1.0.tgz",
+      "integrity": "sha512-+zopwDy7DNknmwPQplem5lAZX/eCOzSvSNNcSKm5eVwTkOBzoktEfXsa9L23J/GIRhxRsaxzkPEhrJEpE2F4Gg==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "agentkeepalive": "^4.1.3",
+        "cacache": "^15.2.0",
+        "http-cache-semantics": "^4.1.0",
+        "http-proxy-agent": "^4.0.1",
+        "https-proxy-agent": "^5.0.0",
+        "is-lambda": "^1.0.1",
+        "lru-cache": "^6.0.0",
+        "minipass": "^3.1.3",
+        "minipass-collect": "^1.0.2",
+        "minipass-fetch": "^1.3.2",
+        "minipass-flush": "^1.0.5",
+        "minipass-pipeline": "^1.2.4",
+        "negotiator": "^0.6.2",
+        "promise-retry": "^2.0.1",
+        "socks-proxy-agent": "^6.0.0",
+        "ssri": "^8.0.0"
+      },
+      "engines": {
+        "node": ">= 10"
+      }
+    },
+    "node_modules/make-fetch-happen/node_modules/lru-cache": {
+      "version": "6.0.0",
+      "resolved": "https://registry.npmjs.org/lru-cache/-/lru-cache-6.0.0.tgz",
+      "integrity": "sha512-Jo6dJ04CmSjuznwJSS3pUeWmd/H0ffTlkXXgwZi+eq1UCmqQwCh+eLsYOYCwY991i2Fah4h1BEMCx4qThGbsiA==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "yallist": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/make-fetch-happen/node_modules/yallist": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/yallist/-/yallist-4.0.0.tgz",
+      "integrity": "sha512-3wdGidZyq5PB084XLES5TpOSRA3wjXAlIWMhum2kRcv/41Sn2emQ0dycQW4uZXLejwKvg6EsvbdlVL+FYEct7A==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/makeerror": {
       "version": "1.0.12",
       "resolved": "https://registry.npmjs.org/makeerror/-/makeerror-1.0.12.tgz",
@@ -7062,6 +7887,16 @@
       "integrity": "sha512-ocnPZQLNpvbedwTy9kNrQEsknEfgvcLMvOtz3sFeWApDq1MXH1TqkCIx58xlpESsfwQOnuBO9beyQuNGzVvuhQ==",
       "license": "Apache-2.0"
     },
+    "node_modules/math-intrinsics": {
+      "version": "1.1.0",
+      "resolved": "https://registry.npmjs.org/math-intrinsics/-/math-intrinsics-1.1.0.tgz",
+      "integrity": "sha512-/IXtbwEk5HTPyEwyKX6hGkYXxM9nbj64B+ilVJnC/R6B0pH5G4V3b0pVbL7DBj4tkhBAppbQUlf6F6Xl9LHu1g==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/memoize-one": {
       "version": "5.2.1",
       "resolved": "https://registry.npmjs.org/memoize-one/-/memoize-one-5.2.1.tgz",
@@ -7582,8 +8417,21 @@
         "node": ">=6"
       }
     },
-    "node_modules/min-indent": {
-      "version": "1.0.1",
+    "node_modules/mimic-response": {
+      "version": "3.1.0",
+      "resolved": "https://registry.npmjs.org/mimic-response/-/mimic-response-3.1.0.tgz",
+      "integrity": "sha512-z0yWI+4FDrrweS8Zmt4Ej5HdJmky15+L2e6Wgn3+iK5fWzb6T3fhNFq2+MeTRb064c6Wr4N/wv0DzQTjNzHNGQ==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">=10"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/sindresorhus"
+      }
+    },
+    "node_modules/min-indent": {
+      "version": "1.0.1",
       "resolved": "https://registry.npmjs.org/min-indent/-/min-indent-1.0.1.tgz",
       "integrity": "sha512-I9jwMn07Sy/IwOj3zVkVik2JTvgpaykDZEigL6Rx6N9LbMywwUSMtxET+7lVoDLLd3O3IXwJwvuuns8UB/HeAg==",
       "dev": true,
@@ -7613,6 +8461,117 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/minipass": {
+      "version": "3.3.6",
+      "resolved": "https://registry.npmjs.org/minipass/-/minipass-3.3.6.tgz",
+      "integrity": "sha512-DxiNidxSEK+tHG6zOIklvNOwm3hvCrbUrdtzY74U6HKTJxvIDfOUL5W5P2Ghd3DTkhhKPYGqeNUIh5qcM4YBfw==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "yallist": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
+    "node_modules/minipass-collect": {
+      "version": "1.0.2",
+      "resolved": "https://registry.npmjs.org/minipass-collect/-/minipass-collect-1.0.2.tgz",
+      "integrity": "sha512-6T6lH0H8OG9kITm/Jm6tdooIbogG9e0tLgpY6mphXSm/A9u8Nq1ryBG+Qspiub9LjWlBPsPS3tWQ/Botq4FdxA==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0"
+      },
+      "engines": {
+        "node": ">= 8"
+      }
+    },
+    "node_modules/minipass-fetch": {
+      "version": "1.4.1",
+      "resolved": "https://registry.npmjs.org/minipass-fetch/-/minipass-fetch-1.4.1.tgz",
+      "integrity": "sha512-CGH1eblLq26Y15+Azk7ey4xh0J/XfJfrCox5LDJiKqI2Q2iwOLOKrlmIaODiSQS8d18jalF6y2K2ePUm0CmShw==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.1.0",
+        "minipass-sized": "^1.0.3",
+        "minizlib": "^2.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      },
+      "optionalDependencies": {
+        "encoding": "^0.1.12"
+      }
+    },
+    "node_modules/minipass-flush": {
+      "version": "1.0.7",
+      "resolved": "https://registry.npmjs.org/minipass-flush/-/minipass-flush-1.0.7.tgz",
+      "integrity": "sha512-TbqTz9cUwWyHS2Dy89P3ocAGUGxKjjLuR9z8w4WUTGAVgEj17/4nhgo2Du56i0Fm3Pm30g4iA8Lcqctc76jCzA==",
+      "license": "BlueOak-1.0.0",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0"
+      },
+      "engines": {
+        "node": ">= 8"
+      }
+    },
+    "node_modules/minipass-pipeline": {
+      "version": "1.2.4",
+      "resolved": "https://registry.npmjs.org/minipass-pipeline/-/minipass-pipeline-1.2.4.tgz",
+      "integrity": "sha512-xuIq7cIOt09RPRJ19gdi4b+RiNvDFYe5JH+ggNvBqGqpQXcru3PcRmOZuHBKWK1Txf9+cQ+HMVN4d6z46LZP7A==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
+    "node_modules/minipass-sized": {
+      "version": "1.0.3",
+      "resolved": "https://registry.npmjs.org/minipass-sized/-/minipass-sized-1.0.3.tgz",
+      "integrity": "sha512-MbkQQ2CTiBMlA2Dm/5cY+9SWFEN8pzzOXi6rlM5Xxq0Yqbda5ZQy9sU75a673FE9ZK0Zsbr6Y5iP6u9nktfg2g==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
+    "node_modules/minipass/node_modules/yallist": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/yallist/-/yallist-4.0.0.tgz",
+      "integrity": "sha512-3wdGidZyq5PB084XLES5TpOSRA3wjXAlIWMhum2kRcv/41Sn2emQ0dycQW4uZXLejwKvg6EsvbdlVL+FYEct7A==",
+      "license": "ISC",
+      "optional": true
+    },
+    "node_modules/minizlib": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/minizlib/-/minizlib-2.1.2.tgz",
+      "integrity": "sha512-bAxsR8BVfj60DWXHE3u30oHzfl4G7khkSuPW+qvpd7jFRHm7dLxOjUk1EHACJ/hxLY8phGJ0YhYHZo7jil7Qdg==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.0.0",
+        "yallist": "^4.0.0"
+      },
+      "engines": {
+        "node": ">= 8"
+      }
+    },
+    "node_modules/minizlib/node_modules/yallist": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/yallist/-/yallist-4.0.0.tgz",
+      "integrity": "sha512-3wdGidZyq5PB084XLES5TpOSRA3wjXAlIWMhum2kRcv/41Sn2emQ0dycQW4uZXLejwKvg6EsvbdlVL+FYEct7A==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/mkdirp": {
       "version": "0.5.6",
       "resolved": "https://registry.npmjs.org/mkdirp/-/mkdirp-0.5.6.tgz",
@@ -7625,12 +8584,26 @@
         "mkdirp": "bin/cmd.js"
       }
     },
+    "node_modules/mkdirp-classic": {
+      "version": "0.5.3",
+      "resolved": "https://registry.npmjs.org/mkdirp-classic/-/mkdirp-classic-0.5.3.tgz",
+      "integrity": "sha512-gKLcREMhtuZRwRAfqP3RFW+TK4JqApVBtOIftVgjuABpAtpxhPGaDcfvbhNvD0B8iD1oUr/txX35NjcaY6Ns/A==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/ms": {
       "version": "2.1.3",
       "resolved": "https://registry.npmjs.org/ms/-/ms-2.1.3.tgz",
       "integrity": "sha512-6FlzubTLZG3J2a/NVCAleEhjzq5oxgHyaCU9yYXvcLsvoVaHJq/s5xXI6/XXP6tz7R9xAOtHnSO/tXtF3WRTlA==",
       "license": "MIT"
     },
+    "node_modules/napi-build-utils": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/napi-build-utils/-/napi-build-utils-2.0.0.tgz",
+      "integrity": "sha512-GEbrYkbfF7MoNaoh2iGG84Mnf/WZfB0GdGEsM8wz7Expx/LlWf5U8t9nvJKXSp3qr5IsEbK04cBGhol/KwOsWA==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/natural-compare": {
       "version": "1.4.0",
       "resolved": "https://registry.npmjs.org/natural-compare/-/natural-compare-1.4.0.tgz",
@@ -7664,6 +8637,39 @@
         "node": ">=12.0.0"
       }
     },
+    "node_modules/node-abi": {
+      "version": "3.95.0",
+      "resolved": "https://registry.npmjs.org/node-abi/-/node-abi-3.95.0.tgz",
+      "integrity": "sha512-T9iGctuocf0qIWFFOTxPzjT5q0SILqaBYXt272tlBHvTKC5+3JnkMirLxNJNkXHtFyBjU2Jx+NL4Zipr0B/c6Q==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "semver": "^7.3.5"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/node-abi/node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "license": "ISC",
+      "optional": true,
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/node-addon-api": {
+      "version": "7.1.1",
+      "resolved": "https://registry.npmjs.org/node-addon-api/-/node-addon-api-7.1.1.tgz",
+      "integrity": "sha512-5m3bsyrjFWE1xf7nz7YXdN4udnVtXK6/Yfgn5qnahL6bCkf2yKt4k3nuTKAtT4r3IG8JNR2ncsIMdZuAzJjHQQ==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/node-dir": {
       "version": "0.1.17",
       "resolved": "https://registry.npmjs.org/node-dir/-/node-dir-0.1.17.tgz",
@@ -7705,6 +8711,44 @@
         "node": ">= 6.13.0"
       }
     },
+    "node_modules/node-gyp": {
+      "version": "8.4.1",
+      "resolved": "https://registry.npmjs.org/node-gyp/-/node-gyp-8.4.1.tgz",
+      "integrity": "sha512-olTJRgUtAb/hOXG0E93wZDs5YiJlgbXxTwQAFHyNlRsXQnYzUaF2aGgujZbw+hR8aF4ZG/rST57bWMWD16jr9w==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "env-paths": "^2.2.0",
+        "glob": "^7.1.4",
+        "graceful-fs": "^4.2.6",
+        "make-fetch-happen": "^9.1.0",
+        "nopt": "^5.0.0",
+        "npmlog": "^6.0.0",
+        "rimraf": "^3.0.2",
+        "semver": "^7.3.5",
+        "tar": "^6.1.2",
+        "which": "^2.0.2"
+      },
+      "bin": {
+        "node-gyp": "bin/node-gyp.js"
+      },
+      "engines": {
+        "node": ">= 10.12.0"
+      }
+    },
+    "node_modules/node-gyp/node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "license": "ISC",
+      "optional": true,
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
     "node_modules/node-int64": {
       "version": "0.4.0",
       "resolved": "https://registry.npmjs.org/node-int64/-/node-int64-0.4.0.tgz",
@@ -7734,6 +8778,28 @@
         "url": "https://github.com/sponsors/antelle"
       }
     },
+    "node_modules/noop-fn": {
+      "version": "1.0.0",
+      "resolved": "https://registry.npmjs.org/noop-fn/-/noop-fn-1.0.0.tgz",
+      "integrity": "sha512-pQ8vODlgXt2e7A3mIbFDlizkr46r75V+BJxVAyat8Jl7YmI513gG5cfyRL0FedKraoZ+VAouI1h4/IWpus5pcQ==",
+      "license": "MIT"
+    },
+    "node_modules/nopt": {
+      "version": "5.0.0",
+      "resolved": "https://registry.npmjs.org/nopt/-/nopt-5.0.0.tgz",
+      "integrity": "sha512-Tbj67rffqceeLpcRXrT7vKAN8CwfPeIBgM7E6iBkmKLV7bEMwpGgYLGv0jACUsECaa/vuxP0IjEont6umdMgtQ==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "abbrev": "1"
+      },
+      "bin": {
+        "nopt": "bin/nopt.js"
+      },
+      "engines": {
+        "node": ">=6"
+      }
+    },
     "node_modules/normalize-path": {
       "version": "3.0.0",
       "resolved": "https://registry.npmjs.org/normalize-path/-/normalize-path-3.0.0.tgz",
@@ -7755,6 +8821,23 @@
         "node": ">=8"
       }
     },
+    "node_modules/npmlog": {
+      "version": "6.0.2",
+      "resolved": "https://registry.npmjs.org/npmlog/-/npmlog-6.0.2.tgz",
+      "integrity": "sha512-/vBvz5Jfr9dT/aFWd0FIRf+T/Q2WBsLENygUaFUqstqsycmZAP/t5BvFJTK0viFmSUxiUKTUplWy5vt+rvKIxg==",
+      "deprecated": "This package is no longer supported.",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "are-we-there-yet": "^3.0.0",
+        "console-control-strings": "^1.1.0",
+        "gauge": "^4.0.3",
+        "set-blocking": "^2.0.0"
+      },
+      "engines": {
+        "node": "^12.13.0 || ^14.15.0 || >=16.0.0"
+      }
+    },
     "node_modules/nullthrows": {
       "version": "1.1.1",
       "resolved": "https://registry.npmjs.org/nullthrows/-/nullthrows-1.1.1.tgz",
@@ -7783,6 +8866,16 @@
         "node": ">=0.10.0"
       }
     },
+    "node_modules/object-keys": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/object-keys/-/object-keys-1.1.1.tgz",
+      "integrity": "sha512-NuAESUOUMrlIXOfHKzD6bpPu3tYt3xvjNdRIQ+FeT0lNb4K8WR70CaDxhuNguS2XG+GjkyMwOzsN5ZktImfhLA==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/on-finished": {
       "version": "2.3.0",
       "resolved": "https://registry.npmjs.org/on-finished/-/on-finished-2.3.0.tgz",
@@ -7879,6 +8972,16 @@
         "node": ">=8"
       }
     },
+    "node_modules/os-tmpdir": {
+      "version": "1.0.2",
+      "resolved": "https://registry.npmjs.org/os-tmpdir/-/os-tmpdir-1.0.2.tgz",
+      "integrity": "sha512-D2FR03Vir7FIu45XBY20mTb+/ZSWB00sjU9jdQXt83gDrI4Ztz5Fs7/yy74g2N5SVQY4xY1qDr4rNddwYRVX0g==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/p-limit": {
       "version": "3.1.0",
       "resolved": "https://registry.npmjs.org/p-limit/-/p-limit-3.1.0.tgz",
@@ -7911,6 +9014,22 @@
         "url": "https://github.com/sponsors/sindresorhus"
       }
     },
+    "node_modules/p-map": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/p-map/-/p-map-4.0.0.tgz",
+      "integrity": "sha512-/bjOqmgETBYB5BoEeGVea8dmvHb2m9GLy1E9W43yeyfP6QQCZGFNa+XRceJEuDB6zqr+gKpIAmlLebMpykw/MQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "aggregate-error": "^3.0.0"
+      },
+      "engines": {
+        "node": ">=10"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/sindresorhus"
+      }
+    },
     "node_modules/p-try": {
       "version": "2.2.0",
       "resolved": "https://registry.npmjs.org/p-try/-/p-try-2.2.0.tgz",
@@ -7961,6 +9080,143 @@
         "node": ">= 0.8"
       }
     },
+    "node_modules/patch-package": {
+      "version": "8.0.0",
+      "resolved": "https://registry.npmjs.org/patch-package/-/patch-package-8.0.0.tgz",
+      "integrity": "sha512-da8BVIhzjtgScwDJ2TtKsfT5JFWz1hYoBl9rUQ1f38MC2HwnEIkK8VN3dKMKcP7P7bvvgzNDbfNHtx3MsQb5vA==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "@yarnpkg/lockfile": "^1.1.0",
+        "chalk": "^4.1.2",
+        "ci-info": "^3.7.0",
+        "cross-spawn": "^7.0.3",
+        "find-yarn-workspace-root": "^2.0.0",
+        "fs-extra": "^9.0.0",
+        "json-stable-stringify": "^1.0.2",
+        "klaw-sync": "^6.0.0",
+        "minimist": "^1.2.6",
+        "open": "^7.4.2",
+        "rimraf": "^2.6.3",
+        "semver": "^7.5.3",
+        "slash": "^2.0.0",
+        "tmp": "^0.0.33",
+        "yaml": "^2.2.2"
+      },
+      "bin": {
+        "patch-package": "index.js"
+      },
+      "engines": {
+        "node": ">=14",
+        "npm": ">5"
+      }
+    },
+    "node_modules/patch-package/node_modules/fs-extra": {
+      "version": "9.1.0",
+      "resolved": "https://registry.npmjs.org/fs-extra/-/fs-extra-9.1.0.tgz",
+      "integrity": "sha512-hcg3ZmepS30/7BSFqRvoo3DOMQu7IjqxO5nCDt+zM9XWjb33Wg7ziNT+Qvqbuc3+gWpzO02JubVyk2G4Zvo1OQ==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "at-least-node": "^1.0.0",
+        "graceful-fs": "^4.2.0",
+        "jsonfile": "^6.0.1",
+        "universalify": "^2.0.0"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/patch-package/node_modules/is-wsl": {
+      "version": "2.2.0",
+      "resolved": "https://registry.npmjs.org/is-wsl/-/is-wsl-2.2.0.tgz",
+      "integrity": "sha512-fKzAra0rGJUUBwGBgNkHZuToZcn+TtXHpeCgmkMJMMYx1sQDYaCSyjJBSCa2nH1DGm7s3n1oBnohoVTBaN7Lww==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "is-docker": "^2.0.0"
+      },
+      "engines": {
+        "node": ">=8"
+      }
+    },
+    "node_modules/patch-package/node_modules/jsonfile": {
+      "version": "6.2.1",
+      "resolved": "https://registry.npmjs.org/jsonfile/-/jsonfile-6.2.1.tgz",
+      "integrity": "sha512-zwOTdL3rFQ/lRdBnntKVOX6k5cKJwEc1HdilT71BWEu7J41gXIB2MRp+vxduPSwZJPWBxEzv4yH1wYLJGUHX4Q==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "universalify": "^2.0.0"
+      },
+      "optionalDependencies": {
+        "graceful-fs": "^4.1.6"
+      }
+    },
+    "node_modules/patch-package/node_modules/open": {
+      "version": "7.4.2",
+      "resolved": "https://registry.npmjs.org/open/-/open-7.4.2.tgz",
+      "integrity": "sha512-MVHddDVweXZF3awtlAS+6pgKLlm/JgxZ90+/NBurBoQctVOOB/zDdVjcyPzQ+0laDGbsWgrRkflI65sQeOgT9Q==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "is-docker": "^2.0.0",
+        "is-wsl": "^2.1.1"
+      },
+      "engines": {
+        "node": ">=8"
+      },
+      "funding": {
+        "url": "https://github.com/sponsors/sindresorhus"
+      }
+    },
+    "node_modules/patch-package/node_modules/rimraf": {
+      "version": "2.7.1",
+      "resolved": "https://registry.npmjs.org/rimraf/-/rimraf-2.7.1.tgz",
+      "integrity": "sha512-uWjbaKIK3T1OSVptzX7Nl6PvQ3qAGtKEtVRjRuazjfL3Bx5eI409VZSqgND+4UNnmzLVdPj9FqFJNPqBZFve4w==",
+      "deprecated": "Rimraf versions prior to v4 are no longer supported",
+      "dev": true,
+      "license": "ISC",
+      "dependencies": {
+        "glob": "^7.1.3"
+      },
+      "bin": {
+        "rimraf": "bin.js"
+      }
+    },
+    "node_modules/patch-package/node_modules/semver": {
+      "version": "7.8.5",
+      "resolved": "https://registry.npmjs.org/semver/-/semver-7.8.5.tgz",
+      "integrity": "sha512-Y7/KDsb8LjooZpwaqGyulO6DQlksgCncchHGk+sZIY4SBvUocMBEFH5Ur1fI4dV+Jvl0w6cjvucaIi40puRioA==",
+      "dev": true,
+      "license": "ISC",
+      "bin": {
+        "semver": "bin/semver.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/patch-package/node_modules/slash": {
+      "version": "2.0.0",
+      "resolved": "https://registry.npmjs.org/slash/-/slash-2.0.0.tgz",
+      "integrity": "sha512-ZYKh3Wh2z1PpEXWr0MpSBZ0V6mZHAQfYevttO11c51CaWjGTaadiKZ+wVt1PbMlDV5qhMFslpZCemhwOK7C89A==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">=6"
+      }
+    },
+    "node_modules/patch-package/node_modules/universalify": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/universalify/-/universalify-2.0.1.tgz",
+      "integrity": "sha512-gptHNQghINnc/vTGIk0SOFGFNXw7JVrlRUtConJRlvaw6DuX0wO5Jeko9sWrMBhh+PsYAZ7oXAiOnf/UKogyiw==",
+      "dev": true,
+      "license": "MIT",
+      "engines": {
+        "node": ">= 10.0.0"
+      }
+    },
     "node_modules/path-exists": {
       "version": "4.0.0",
       "resolved": "https://registry.npmjs.org/path-exists/-/path-exists-4.0.0.tgz",
@@ -8099,6 +9355,34 @@
         "node": ">=8"
       }
     },
+    "node_modules/prebuild-install": {
+      "version": "7.1.3",
+      "resolved": "https://registry.npmjs.org/prebuild-install/-/prebuild-install-7.1.3.tgz",
+      "integrity": "sha512-8Mf2cbV7x1cXPUILADGI3wuhfqWvtiLA1iclTDbFRZkgRQS0NqsPZphna9V+HyTEadheuPmjaJMsbzKQFOzLug==",
+      "deprecated": "No longer maintained. Please contact the author of the relevant native addon; alternatives are available.",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "detect-libc": "^2.0.0",
+        "expand-template": "^2.0.3",
+        "github-from-package": "0.0.0",
+        "minimist": "^1.2.3",
+        "mkdirp-classic": "^0.5.3",
+        "napi-build-utils": "^2.0.0",
+        "node-abi": "^3.3.0",
+        "pump": "^3.0.0",
+        "rc": "^1.2.7",
+        "simple-get": "^4.0.0",
+        "tar-fs": "^2.0.0",
+        "tunnel-agent": "^0.6.0"
+      },
+      "bin": {
+        "prebuild-install": "bin.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
     "node_modules/pretty-format": {
       "version": "26.6.2",
       "resolved": "https://registry.npmjs.org/pretty-format/-/pretty-format-26.6.2.tgz",
@@ -8151,6 +9435,27 @@
         "asap": "~2.0.6"
       }
     },
+    "node_modules/promise-inflight": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/promise-inflight/-/promise-inflight-1.0.1.tgz",
+      "integrity": "sha512-6zWPyEOFaQBJYcGMHBKTKJ3u6TBsnMFOIZSa6ce1e/ZrrsOlnHRHbabMjLiBYKp+n44X9eUI6VUPaukCXHuG4g==",
+      "license": "ISC",
+      "optional": true
+    },
+    "node_modules/promise-retry": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/promise-retry/-/promise-retry-2.0.1.tgz",
+      "integrity": "sha512-y+WKFlBR8BGXnsNlIHFGPZmyDf3DFMoLhaflAnyZgV6rG6xu+JwesTo2Q9R6XwYmtmwAFCkAk3e35jEdoeh/3g==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "err-code": "^2.0.2",
+        "retry": "^0.12.0"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
     "node_modules/prompts": {
       "version": "2.4.2",
       "resolved": "https://registry.npmjs.org/prompts/-/prompts-2.4.2.tgz",
@@ -8165,6 +9470,17 @@
         "node": ">= 6"
       }
     },
+    "node_modules/pump": {
+      "version": "3.0.4",
+      "resolved": "https://registry.npmjs.org/pump/-/pump-3.0.4.tgz",
+      "integrity": "sha512-VS7sjc6KR7e1ukRFhQSY5LM2uBWAUPiOPa/A3mkKmiMwSmRFUITt0xuj+/lesgnCv+dPIEYlkzrcyXgquIHMcA==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "end-of-stream": "^1.1.0",
+        "once": "^1.3.1"
+      }
+    },
     "node_modules/pure-rand": {
       "version": "6.1.0",
       "resolved": "https://registry.npmjs.org/pure-rand/-/pure-rand-6.1.0.tgz",
@@ -8221,6 +9537,32 @@
         "node": ">= 0.6"
       }
     },
+    "node_modules/rc": {
+      "version": "1.2.8",
+      "resolved": "https://registry.npmjs.org/rc/-/rc-1.2.8.tgz",
+      "integrity": "sha512-y3bGgqKj3QBdxLbLkomlohkvsA8gdAiUQlSBJnBhfn+BPxg4bc62d8TcBW15wavDfgexCgccckhcZvywyQYPOw==",
+      "license": "(BSD-2-Clause OR MIT OR Apache-2.0)",
+      "optional": true,
+      "dependencies": {
+        "deep-extend": "^0.6.0",
+        "ini": "~1.3.0",
+        "minimist": "^1.2.0",
+        "strip-json-comments": "~2.0.1"
+      },
+      "bin": {
+        "rc": "cli.js"
+      }
+    },
+    "node_modules/rc/node_modules/strip-json-comments": {
+      "version": "2.0.1",
+      "resolved": "https://registry.npmjs.org/strip-json-comments/-/strip-json-comments-2.0.1.tgz",
+      "integrity": "sha512-4gB8na07fecVVkOI6Rs4e7T6NOTki5EmL7TUduTs6bu3EdnSycntVJ4re8kgZA+wx9IueI2Y11bfbgwtzuE0KQ==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">=0.10.0"
+      }
+    },
     "node_modules/react": {
       "version": "18.3.1",
       "resolved": "https://registry.npmjs.org/react/-/react-18.3.1.tgz",
@@ -8332,6 +9674,20 @@
         }
       }
     },
+    "node_modules/react-native-sqlite-2": {
+      "version": "3.6.3",
+      "resolved": "https://registry.npmjs.org/react-native-sqlite-2/-/react-native-sqlite-2-3.6.3.tgz",
+      "integrity": "sha512-cA+7npoem+JMumU9kimEE7soQMSdzjj544XcYzzeKDcerfiSfydt+Ife8byvDx/VeHxL4t6MrJ7qjvQtvXFeOA==",
+      "license": "Apache-2.0",
+      "dependencies": {
+        "lodash.map": "^4.6.0",
+        "lodash.zipobject": "^4.1.3",
+        "websql": "^2.0.3"
+      },
+      "peerDependencies": {
+        "react-native": ">= 0.60.0"
+      }
+    },
     "node_modules/react-native/node_modules/@react-native/virtualized-lists": {
       "version": "0.76.9",
       "resolved": "https://registry.npmjs.org/@react-native/virtualized-lists/-/virtualized-lists-0.76.9.tgz",
@@ -8670,6 +10026,16 @@
         "node": ">=8"
       }
     },
+    "node_modules/retry": {
+      "version": "0.12.0",
+      "resolved": "https://registry.npmjs.org/retry/-/retry-0.12.0.tgz",
+      "integrity": "sha512-9LkiTwjUh6rT555DtE9rTX+BKByPfrMzEAtnlEtdEwr3Nkffwiihqe2bWADg+OQRjt9gl6ICdmB/ZFDCGAtSow==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">= 4"
+      }
+    },
     "node_modules/reusify": {
       "version": "1.1.0",
       "resolved": "https://registry.npmjs.org/reusify/-/reusify-1.1.0.tgz",
@@ -8742,6 +10108,13 @@
       ],
       "license": "MIT"
     },
+    "node_modules/safer-buffer": {
+      "version": "2.1.2",
+      "resolved": "https://registry.npmjs.org/safer-buffer/-/safer-buffer-2.1.2.tgz",
+      "integrity": "sha512-YZo3K82SD7Riyi0E1EQPojLz7kpepnSQI9IyPbHHg1XXXevb5dJI7tpyN2ADxGcQbHG7vcyRHk0cbwqcQriUtg==",
+      "license": "MIT",
+      "optional": true
+    },
     "node_modules/scheduler": {
       "version": "0.24.0-canary-efb381bbf-20230505",
       "resolved": "https://registry.npmjs.org/scheduler/-/scheduler-0.24.0-canary-efb381bbf-20230505.tgz",
@@ -8894,6 +10267,24 @@
       "devOptional": true,
       "license": "ISC"
     },
+    "node_modules/set-function-length": {
+      "version": "1.2.2",
+      "resolved": "https://registry.npmjs.org/set-function-length/-/set-function-length-1.2.2.tgz",
+      "integrity": "sha512-pgRc4hJ4/sNjWCSS9AmnS40x3bNMDTknHgL5UaMBTMyJnU90EgWh1Rz+MC9eFu4BuN/UwZjKQuY/1v3rM7HMfg==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "define-data-property": "^1.1.4",
+        "es-errors": "^1.3.0",
+        "function-bind": "^1.1.2",
+        "get-intrinsic": "^1.2.4",
+        "gopd": "^1.0.1",
+        "has-property-descriptors": "^1.0.2"
+      },
+      "engines": {
+        "node": ">= 0.4"
+      }
+    },
     "node_modules/setprototypeof": {
       "version": "1.2.0",
       "resolved": "https://registry.npmjs.org/setprototypeof/-/setprototypeof-1.2.0.tgz",
@@ -8951,6 +10342,53 @@
       "integrity": "sha512-wnD2ZE+l+SPC/uoS0vXeE9L1+0wuaMqKlfz9AMUo38JsyLSBWSFcHR1Rri62LZc12vLr1gb3jl7iwQhgwpAbGQ==",
       "license": "ISC"
     },
+    "node_modules/simple-concat": {
+      "version": "1.0.1",
+      "resolved": "https://registry.npmjs.org/simple-concat/-/simple-concat-1.0.1.tgz",
+      "integrity": "sha512-cSFtAPtRhljv69IK0hTVZQ+OfE9nePi/rtJmw5UjHeVyVroEqJXP1sFztKUy1qU+xvz3u/sfYJLa947b7nAN2Q==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/feross"
+        },
+        {
+          "type": "patreon",
+          "url": "https://www.patreon.com/feross"
+        },
+        {
+          "type": "consulting",
+          "url": "https://feross.org/support"
+        }
+      ],
+      "license": "MIT",
+      "optional": true
+    },
+    "node_modules/simple-get": {
+      "version": "4.0.1",
+      "resolved": "https://registry.npmjs.org/simple-get/-/simple-get-4.0.1.tgz",
+      "integrity": "sha512-brv7p5WgH0jmQJr1ZDDfKDOSeWWg+OVypG99A/5vYGPqJ6pxiaHLy8nxtFjBA7oMa01ebA9gfh1uMCFqOuXxvA==",
+      "funding": [
+        {
+          "type": "github",
+          "url": "https://github.com/sponsors/feross"
+        },
+        {
+          "type": "patreon",
+          "url": "https://www.patreon.com/feross"
+        },
+        {
+          "type": "consulting",
+          "url": "https://feross.org/support"
+        }
+      ],
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "decompress-response": "^6.0.0",
+        "once": "^1.3.1",
+        "simple-concat": "^1.0.0"
+      }
+    },
     "node_modules/sisteransi": {
       "version": "1.0.5",
       "resolved": "https://registry.npmjs.org/sisteransi/-/sisteransi-1.0.5.tgz",
@@ -9012,6 +10450,47 @@
       "devOptional": true,
       "license": "MIT"
     },
+    "node_modules/smart-buffer": {
+      "version": "4.2.0",
+      "resolved": "https://registry.npmjs.org/smart-buffer/-/smart-buffer-4.2.0.tgz",
+      "integrity": "sha512-94hK0Hh8rPqQl2xXc3HsaBoOXKV20MToPkcXvwbISWLEs+64sBq5kFgn2kJDHb1Pry9yrP0dxrCI9RRci7RXKg==",
+      "license": "MIT",
+      "optional": true,
+      "engines": {
+        "node": ">= 6.0.0",
+        "npm": ">= 3.0.0"
+      }
+    },
+    "node_modules/socks": {
+      "version": "2.8.9",
+      "resolved": "https://registry.npmjs.org/socks/-/socks-2.8.9.tgz",
+      "integrity": "sha512-LJhUYUvItdQ0LkJTmPeaEObWXAqFyfmP85x0tch/ez9cahmhlBBLbIqDFnvBnUJGagb0JbIQrkBs1wJ+yRYpEw==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "ip-address": "^10.1.1",
+        "smart-buffer": "^4.2.0"
+      },
+      "engines": {
+        "node": ">= 10.0.0",
+        "npm": ">= 3.0.0"
+      }
+    },
+    "node_modules/socks-proxy-agent": {
+      "version": "6.2.1",
+      "resolved": "https://registry.npmjs.org/socks-proxy-agent/-/socks-proxy-agent-6.2.1.tgz",
+      "integrity": "sha512-a6KW9G+6B3nWZ1yB8G7pJwL3ggLy1uTzKAgCb7ttblwqdz9fMGJUuTy3uFzEP48FAs9FLILlmzDlE2JJhVQaXQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "agent-base": "^6.0.2",
+        "debug": "^4.3.3",
+        "socks": "^2.6.2"
+      },
+      "engines": {
+        "node": ">= 10"
+      }
+    },
     "node_modules/source-map": {
       "version": "0.6.1",
       "resolved": "https://registry.npmjs.org/source-map/-/source-map-0.6.1.tgz",
@@ -9038,6 +10517,44 @@
       "integrity": "sha512-D9cPgkvLlV3t3IzL0D0YLvGA9Ahk4PcvVwUbN0dSGr1aP0Nrt4AEnTUbuGvquEC0mA64Gqt1fzirlRs5ibXx8g==",
       "license": "BSD-3-Clause"
     },
+    "node_modules/sqlite3": {
+      "version": "5.1.7",
+      "resolved": "https://registry.npmjs.org/sqlite3/-/sqlite3-5.1.7.tgz",
+      "integrity": "sha512-GGIyOiFaG+TUra3JIfkI/zGP8yZYLPQ0pl1bH+ODjiX57sPhrLU5sQJn1y9bDKZUFYkX1crlrPfSYt0BKKdkog==",
+      "hasInstallScript": true,
+      "license": "BSD-3-Clause",
+      "optional": true,
+      "dependencies": {
+        "bindings": "^1.5.0",
+        "node-addon-api": "^7.0.0",
+        "prebuild-install": "^7.1.1",
+        "tar": "^6.1.11"
+      },
+      "optionalDependencies": {
+        "node-gyp": "8.x"
+      },
+      "peerDependencies": {
+        "node-gyp": "8.x"
+      },
+      "peerDependenciesMeta": {
+        "node-gyp": {
+          "optional": true
+        }
+      }
+    },
+    "node_modules/ssri": {
+      "version": "8.0.1",
+      "resolved": "https://registry.npmjs.org/ssri/-/ssri-8.0.1.tgz",
+      "integrity": "sha512-97qShzy1AiyxvPNIkLWoGua7xoQzzPjQ0HAH4B0rWKo7SZ6USuPcrUiAFrws0UH8RrbWmgq3LMTObhPIHbbBeQ==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "minipass": "^3.1.1"
+      },
+      "engines": {
+        "node": ">= 8"
+      }
+    },
     "node_modules/stack-utils": {
       "version": "2.0.6",
       "resolved": "https://registry.npmjs.org/stack-utils/-/stack-utils-2.0.6.tgz",
@@ -9280,6 +10797,92 @@
         "url": "https://github.com/sponsors/ljharb"
       }
     },
+    "node_modules/tar": {
+      "version": "6.2.1",
+      "resolved": "https://registry.npmjs.org/tar/-/tar-6.2.1.tgz",
+      "integrity": "sha512-DZ4yORTwrbTj/7MZYq2w+/ZFdI6OZ/f9SFHR+71gIVUZhOQPHzVCLpvRnPgyaMpfWxxk/4ONva3GQSyNIKRv6A==",
+      "deprecated": "Old versions of tar are not supported, and contain widely publicized security vulnerabilities, which have been fixed in the current version. Please update. Support for old versions may be purchased (at exorbitant rates) by contacting i@izs.me",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "chownr": "^2.0.0",
+        "fs-minipass": "^2.0.0",
+        "minipass": "^5.0.0",
+        "minizlib": "^2.1.1",
+        "mkdirp": "^1.0.3",
+        "yallist": "^4.0.0"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/tar-fs": {
+      "version": "2.1.5",
+      "resolved": "https://registry.npmjs.org/tar-fs/-/tar-fs-2.1.5.tgz",
+      "integrity": "sha512-OboTd8mmMhZDNPV+UjQcK9yKAatXu2aJ+r1w4im1Otd4M4fl2hwvdoXUxIYHFTHWK/3y3FarBP70v3vwmGlOxw==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "chownr": "^1.1.1",
+        "mkdirp-classic": "^0.5.2",
+        "pump": "^3.0.0",
+        "tar-stream": "^2.1.4"
+      }
+    },
+    "node_modules/tar-fs/node_modules/chownr": {
+      "version": "1.1.4",
+      "resolved": "https://registry.npmjs.org/chownr/-/chownr-1.1.4.tgz",
+      "integrity": "sha512-jJ0bqzaylmJtVnNgzTeSOs8DPavpbYgEr/b0YL8/2GO3xJEhInFmhKMUnEJQjZumK7KXGFhUy89PrsJWlakBVg==",
+      "license": "ISC",
+      "optional": true
+    },
+    "node_modules/tar-stream": {
+      "version": "2.2.0",
+      "resolved": "https://registry.npmjs.org/tar-stream/-/tar-stream-2.2.0.tgz",
+      "integrity": "sha512-ujeqbceABgwMZxEJnk2HDY2DlnUZ+9oEcb1KzTVfYHio0UE6dG71n60d8D2I4qNvleWrrXpmjpt7vZeF1LnMZQ==",
+      "license": "MIT",
+      "optional": true,
+      "dependencies": {
+        "bl": "^4.0.3",
+        "end-of-stream": "^1.4.1",
+        "fs-constants": "^1.0.0",
+        "inherits": "^2.0.3",
+        "readable-stream": "^3.1.1"
+      },
+      "engines": {
+        "node": ">=6"
+      }
+    },
+    "node_modules/tar/node_modules/minipass": {
+      "version": "5.0.0",
+      "resolved": "https://registry.npmjs.org/minipass/-/minipass-5.0.0.tgz",
+      "integrity": "sha512-3FnjYuehv9k6ovOEbyOswadCDPX1piCfhV8ncmYtHOjuPwylVWsghTLo7rabjC3Rx5xD4HDx8Wm1xnMF7S5qFQ==",
+      "license": "ISC",
+      "optional": true,
+      "engines": {
+        "node": ">=8"
+      }
+    },
+    "node_modules/tar/node_modules/mkdirp": {
+      "version": "1.0.4",
+      "resolved": "https://registry.npmjs.org/mkdirp/-/mkdirp-1.0.4.tgz",
+      "integrity": "sha512-vVqVZQyf3WLx2Shd0qJ9xuvqgAyKPLAiqITEtqW0oIUjzo3PePDd6fW9iFz30ef7Ysp/oiWqbhszeGWW2T6Gzw==",
+      "license": "MIT",
+      "optional": true,
+      "bin": {
+        "mkdirp": "bin/cmd.js"
+      },
+      "engines": {
+        "node": ">=10"
+      }
+    },
+    "node_modules/tar/node_modules/yallist": {
+      "version": "4.0.0",
+      "resolved": "https://registry.npmjs.org/yallist/-/yallist-4.0.0.tgz",
+      "integrity": "sha512-3wdGidZyq5PB084XLES5TpOSRA3wjXAlIWMhum2kRcv/41Sn2emQ0dycQW4uZXLejwKvg6EsvbdlVL+FYEct7A==",
+      "license": "ISC",
+      "optional": true
+    },
     "node_modules/temp": {
       "version": "0.8.4",
       "resolved": "https://registry.npmjs.org/temp/-/temp-0.8.4.tgz",
@@ -9359,6 +10962,25 @@
       "integrity": "sha512-fcwX4mndzpLQKBS1DVYhGAcYaYt7vsHNIvQV+WXMvnow5cgjPphq5CaayLaGsjRdSCKZFNGt7/GYAuXaNOiYCA==",
       "license": "MIT"
     },
+    "node_modules/tiny-queue": {
+      "version": "0.2.1",
+      "resolved": "https://registry.npmjs.org/tiny-queue/-/tiny-queue-0.2.1.tgz",
+      "integrity": "sha512-EijGsv7kzd9I9g0ByCl6h42BWNGUZrlCSejfrb3AKeHC33SGbASu1VDf5O3rRiiUOhAC9CHdZxFPbZu0HmR70A==",
+      "license": "Apache 2"
+    },
+    "node_modules/tmp": {
+      "version": "0.0.33",
+      "resolved": "https://registry.npmjs.org/tmp/-/tmp-0.0.33.tgz",
+      "integrity": "sha512-jRCJlojKnZ3addtTOjdIqoRuPEKBvNXcGYqzO6zWZX8KfKEpnGY5jfggJQ3EjKuu8D4bJRr0y+cYJFmYbImXGw==",
+      "dev": true,
+      "license": "MIT",
+      "dependencies": {
+        "os-tmpdir": "~1.0.2"
+      },
+      "engines": {
+        "node": ">=0.6.0"
+      }
+    },
     "node_modules/tmpl": {
       "version": "1.0.5",
       "resolved": "https://registry.npmjs.org/tmpl/-/tmpl-1.0.5.tgz",
@@ -9398,6 +11020,19 @@
       "integrity": "sha512-oJFu94HQb+KVduSUQL7wnpmqnfmLsOA/nAh6b6EH0wCEoK0/mPeXU6c3wKDV83MkOuHPRHtSXKKU99IBazS/2w==",
       "license": "0BSD"
     },
+    "node_modules/tunnel-agent": {
+      "version": "0.6.0",
+      "resolved": "https://registry.npmjs.org/tunnel-agent/-/tunnel-agent-0.6.0.tgz",
+      "integrity": "sha512-McnNiV1l8RYeY8tBgEpuodCC1mLUdbSN+CYBL7kJsJNInOP8UjDDEwdk6Mw60vdLLrr5NHKZhMAOSrR2NZuQ+w==",
+      "license": "Apache-2.0",
+      "optional": true,
+      "dependencies": {
+        "safe-buffer": "^5.0.1"
+      },
+      "engines": {
+        "node": "*"
+      }
+    },
     "node_modules/type-detect": {
       "version": "4.0.8",
       "resolved": "https://registry.npmjs.org/type-detect/-/type-detect-4.0.8.tgz",
@@ -9480,6 +11115,26 @@
         "node": ">=4"
       }
     },
+    "node_modules/unique-filename": {
+      "version": "1.1.1",
+      "resolved": "https://registry.npmjs.org/unique-filename/-/unique-filename-1.1.1.tgz",
+      "integrity": "sha512-Vmp0jIp2ln35UTXuryvjzkjGdRyf9b2lTXuSYUiPmzRcl3FDtYqAwOnTJkAngD9SWhnoJzDbTKwaOrZ+STtxNQ==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "unique-slug": "^2.0.0"
+      }
+    },
+    "node_modules/unique-slug": {
+      "version": "2.0.2",
+      "resolved": "https://registry.npmjs.org/unique-slug/-/unique-slug-2.0.2.tgz",
+      "integrity": "sha512-zoWr9ObaxALD3DOPfjPSqxt4fnZiWblxHIgeWqW8x7UqDzEtHEQLzji2cuJYQFCU6KmoJikOYAZlrTHHebjx2w==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "imurmurhash": "^0.1.4"
+      }
+    },
     "node_modules/universalify": {
       "version": "0.1.2",
       "resolved": "https://registry.npmjs.org/universalify/-/universalify-0.1.2.tgz",
@@ -9601,6 +11256,21 @@
       "integrity": "sha512-2JAn3z8AR6rjK8Sm8orRC0h/bcl/DqL7tRPdGZ4I1CjdF+EaMLmYxBHyXuKL849eucPFhvBoxMsflfOb8kxaeQ==",
       "license": "BSD-2-Clause"
     },
+    "node_modules/websql": {
+      "version": "2.0.3",
+      "resolved": "https://registry.npmjs.org/websql/-/websql-2.0.3.tgz",
+      "integrity": "sha512-bSYpuhQ4ODKrWLb6S+9BG2T4AMqHLjCQA9r8UWCapPvTZYXoembz0O14Ga4EAfJuO1wkmFcJjgU/6tzvPfGbmA==",
+      "license": "Apache-2.0",
+      "dependencies": {
+        "argsarray": "^0.0.1",
+        "immediate": "^3.2.2",
+        "noop-fn": "^1.0.0",
+        "tiny-queue": "^0.2.1"
+      },
+      "optionalDependencies": {
+        "sqlite3": "^5.0.2"
+      }
+    },
     "node_modules/whatwg-fetch": {
       "version": "3.6.20",
       "resolved": "https://registry.npmjs.org/whatwg-fetch/-/whatwg-fetch-3.6.20.tgz",
@@ -9639,6 +11309,16 @@
       "devOptional": true,
       "license": "ISC"
     },
+    "node_modules/wide-align": {
+      "version": "1.1.5",
+      "resolved": "https://registry.npmjs.org/wide-align/-/wide-align-1.1.5.tgz",
+      "integrity": "sha512-eDMORYaPNZ4sQIuuYPDHdQvf4gyCF9rEEV/yPxGfwPkRodwEgiMUUXTx/dex+Me0wxx53S+NgUHaP7y3MGlDmg==",
+      "license": "ISC",
+      "optional": true,
+      "dependencies": {
+        "string-width": "^1.0.2 || 2 || 3 || 4"
+      }
+    },
     "node_modules/wrap-ansi": {
       "version": "7.0.0",
       "resolved": "https://registry.npmjs.org/wrap-ansi/-/wrap-ansi-7.0.0.tgz",
diff --git a/package.json b/package.json
index be032f4..eb00a2c 100644
--- a/package.json
+++ b/package.json
@@ -3,12 +3,14 @@
   "version": "0.1.0",
   "private": true,
   "scripts": {
+    "postinstall": "patch-package",
     "test": "jest",
     "typecheck": "tsc --noEmit"
   },
   "dependencies": {
     "react": "18.3.1",
-    "react-native": "0.76.9"
+    "react-native": "0.76.9",
+    "react-native-sqlite-2": "3.6.3"
   },
   "devDependencies": {
     "@babel/core": "7.25.2",
@@ -24,6 +26,7 @@
     "@types/react-test-renderer": "18.3.0",
     "babel-jest": "29.7.0",
     "jest": "29.7.0",
+    "patch-package": "8.0.0",
     "react-test-renderer": "18.3.1",
     "typescript": "5.6.3"
   },
diff --git a/patches/react-native-sqlite-2+3.6.3.patch b/patches/react-native-sqlite-2+3.6.3.patch
new file mode 100644
index 0000000..bb74655
--- /dev/null
+++ b/patches/react-native-sqlite-2+3.6.3.patch
@@ -0,0 +1,25 @@
+diff --git a/node_modules/react-native-sqlite-2/android/build.gradle b/node_modules/react-native-sqlite-2/android/build.gradle
+index 69396b3..74dfd59 100644
+--- a/node_modules/react-native-sqlite-2/android/build.gradle
++++ b/node_modules/react-native-sqlite-2/android/build.gradle
+@@ -22,9 +22,9 @@ def getExtOrIntegerDefault(name) {
+ }
+ 
+ android {
++  namespace 'dog.craftz.sqlite_2'
+   compileSdkVersion getExtOrIntegerDefault('compileSdkVersion')
+   buildToolsVersion getExtOrDefault('buildToolsVersion')
+-  ndkVersion getExtOrDefault('ndkVersion')
+ 
+   defaultConfig {
+     minSdkVersion 21
+diff --git a/node_modules/react-native-sqlite-2/android/src/main/AndroidManifest.xml b/node_modules/react-native-sqlite-2/android/src/main/AndroidManifest.xml
+index 095932c..0a0938a 100644
+--- a/node_modules/react-native-sqlite-2/android/src/main/AndroidManifest.xml
++++ b/node_modules/react-native-sqlite-2/android/src/main/AndroidManifest.xml
+@@ -1,4 +1,3 @@
+-<manifest xmlns:android="http://schemas.android.com/apk/res/android"
+-          package="dog.craftz.sqlite_2">
++<manifest xmlns:android="http://schemas.android.com/apk/res/android">
+ 
+ </manifest>
diff --git a/scripts/verify_m02.py b/scripts/verify_m02.py
new file mode 100644
index 0000000..7579a4f
--- /dev/null
+++ b/scripts/verify_m02.py
@@ -0,0 +1,214 @@
+#!/usr/bin/env python3
+"""External Android UI/process harness. The host survives the target app's death."""
+import argparse
+import json
+import os
+from pathlib import Path
+import re
+import sqlite3
+import subprocess
+import time
+import xml.etree.ElementTree as ET
+
+
+PACKAGE = "com.mse.reactnative"
+
+
+def wait_for_network(read_state, expected):
+    # svc returns before the Settings value necessarily reflects its change.
+    # This wait is cleanup only; it does not change the fixed app scenario.
+    deadline = time.monotonic() + 10
+    actual = read_state()
+    while actual != expected and time.monotonic() < deadline:
+        time.sleep(0.1)
+        actual = read_state()
+    return actual
+
+
+def main():
+    parser = argparse.ArgumentParser()
+    parser.add_argument("--adb", default="adb")
+    parser.add_argument("--serial", default="emulator-5554")
+    parser.add_argument("--apk", required=True)
+    parser.add_argument("--evidence", required=True)
+    parser.add_argument("--expect-loss", action="store_true")
+    args = parser.parse_args()
+    evidence = Path(args.evidence)
+    evidence.mkdir(parents=True, exist_ok=True)
+    commands = []
+
+    def adb(*parts, check=True, binary=False):
+        command = [args.adb, "-s", args.serial, *parts]
+        result = subprocess.run(command, capture_output=True, timeout=90)
+        commands.append({"command": command, "exit": result.returncode,
+                         "stdout": "<binary>" if binary else result.stdout.decode(errors="replace"),
+                         "stderr": result.stderr.decode(errors="replace")})
+        (evidence / "commands.json").write_text(json.dumps(commands, indent=2))
+        if check and result.returncode:
+            raise AssertionError(commands[-1])
+        return result.stdout if binary else result.stdout.decode(errors="replace").strip()
+
+    def snapshot(name=None):
+        adb("shell", "uiautomator", "dump", "/sdcard/mse-m02-ui.xml")
+        xml = adb("exec-out", "cat", "/sdcard/mse-m02-ui.xml")
+        if name:
+            (evidence / f"{name}.xml").write_text(xml)
+            (evidence / f"{name}.png").write_bytes(adb("exec-out", "screencap", "-p", binary=True))
+        return ET.fromstring(xml)
+
+    def find(label, attribute="content-desc"):
+        deadline = time.monotonic() + 20
+        while time.monotonic() < deadline:
+            root = snapshot()
+            for node in root.iter("node"):
+                if node.get(attribute) == label:
+                    return node
+        raise AssertionError(f"Missing {attribute}: {label}")
+
+    def tap(label):
+        node = find(label)
+        x1, y1, x2, y2 = map(int, re.findall(r"\d+", node.get("bounds")))
+        adb("shell", "input", "tap", str((x1 + x2) // 2), str((y1 + y2) // 2))
+
+    def title(label, value):
+        node = find(label)
+        previous = node.get("text", "")
+        tap(label)
+        adb("shell", "input", "keyevent", "KEYCODE_MOVE_END")
+        if previous and previous != "Item title":
+            adb("shell", "input", "keyevent", *(["KEYCODE_DEL"] * len(previous)))
+        adb("shell", "input", "text", value.replace(" ", "%s"))
+        adb("shell", "input", "keyevent", "KEYCODE_BACK")
+
+    def assert_final(root):
+        nodes = list(root.iter("node"))
+        assert any(n.get("content-desc") == "Item count: 1" for n in nodes)
+        assert any(n.get("text") == "Alpha edited" for n in nodes)
+        assert any(n.get("resource-id") == "item-row-item-001" for n in nodes)
+        assert any(n.get("content-desc") == "Mark Alpha edited incomplete" and n.get("checked") == "true" for n in nodes)
+        assert not any(n.get("text") == "Beta" or n.get("content-desc") == "Delete Beta" for n in nodes)
+
+    def database_state(name):
+        # The UI is idle after native COMMIT; capture the database and any WAL.
+        files = adb("shell", "run-as", PACKAGE, "ls", "files").splitlines()
+        path = evidence / f"{name}.db"
+        for suffix in ("", "-wal", "-shm"):
+            source = f"items.db{suffix}"
+            if source in files:
+                Path(str(path) + suffix).write_bytes(adb("exec-out", "run-as", PACKAGE, "cat", f"files/{source}", binary=True))
+        assert path.exists(), "Native database file missing"
+        with sqlite3.connect(f"file:{path}?mode=ro", uri=True) as database:
+            assert database.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
+            assert database.execute("PRAGMA user_version").fetchone()[0] == 1
+            assert [column[1] for column in database.execute("PRAGMA table_info(items)")] == ["id", "title", "completed", "version", "updated_at"]
+            items = [{"id": row[0], "title": row[1], "completed": bool(row[2]), "version": row[3], "updatedAt": row[4]}
+                     for row in database.execute("SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid")]
+            state = {"schema_version": 1, "items": items,
+                     "next_id": database.execute("SELECT next_id FROM local_identity WHERE singleton=1").fetchone()[0]}
+        (evidence / f"{name}.json").write_text(json.dumps(state, indent=2))
+        return state
+
+    def network_state():
+        return {key: adb("shell", "settings", "get", "global", key)
+                for key in ("airplane_mode_on", "wifi_on", "mobile_data")}
+
+    result = {"host_pid": os.getpid(), "serial": args.serial, "expected": "loss" if args.expect_loss else "durability"}
+    network_before = None
+    try:
+        adb("install", "-r", args.apk)
+        adb("shell", "pm", "clear", PACKAGE)
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        find("Item count: 0")
+        title("New item title", "Alpha")
+        tap("Add item")
+        find("Item count: 1")
+        title("New item title", "Beta")
+        tap("Add item")
+        find("Item count: 2")
+        tap("Edit Alpha")
+        title("Edit item title", "Alpha edited")
+        tap("Save title")
+        find("Alpha edited", "text")
+        tap("Mark Alpha edited complete")
+        assert find("Mark Alpha edited incomplete").get("checked") == "true"
+        tap("Mark Alpha edited incomplete")
+        assert find("Mark Alpha edited complete").get("checked") == "false"
+        tap("Mark Alpha edited complete")
+        assert find("Mark Alpha edited incomplete").get("checked") == "true"
+        tap("Delete Beta")
+        find("Item count: 1")
+        if not args.expect_loss:
+            find("Local storage ready")
+        assert_final(snapshot("before-kill"))
+        if not args.expect_loss:
+            result["before_kill"] = database_state("before-kill")
+            before = result["before_kill"]["items"]
+            assert len(before) == 1
+            assert before[0]["id"] == "item-001" and before[0]["title"] == "Alpha edited"
+            assert before[0]["completed"] is True and before[0]["version"] == 5
+            assert isinstance(before[0]["updatedAt"], int) and before[0]["updatedAt"] > 0
+            assert result["before_kill"]["next_id"] == 3
+            network_before = network_state()
+            result["network_before"] = network_before
+            adb("shell", "cmd", "connectivity", "airplane-mode", "enable")
+            adb("shell", "svc", "wifi", "disable")
+            adb("shell", "svc", "data", "disable")
+            result["network_during_restart"] = network_state()
+            assert result["network_during_restart"] == {"airplane_mode_on": "1", "wifi_on": "0", "mobile_data": "0"}
+            (evidence / "offline-connectivity.txt").write_text(adb("shell", "dumpsys", "connectivity"))
+        result["pid_before"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_before"]
+        adb("shell", "am", "force-stop", PACKAGE)
+        result["pid_after_kill"] = adb("shell", "pidof", PACKAGE, check=False)
+        assert not result["pid_after_kill"]
+        adb("shell", "am", "start", "-W", "-n", f"{PACKAGE}/.MainActivity")
+        find("Item count: 0" if args.expect_loss else "Item count: 1")
+        result["pid_after_restart"] = adb("shell", "pidof", PACKAGE)
+        assert result["pid_after_restart"] and result["pid_after_restart"] != result["pid_before"]
+        after = snapshot("after-restart")
+        if args.expect_loss:
+            assert not any(node.get("text") in ("Alpha edited", "Beta") for node in after.iter("node"))
+        else:
+            find("Local storage ready")
+            assert_final(after)
+            result["after_restart"] = database_state("after-restart")
+            assert result["after_restart"] == result["before_kill"], "All five Item fields and identity must survive"
+            # A separate post-acceptance probe ensures a new process cannot reuse
+            # either the surviving identity or the deleted Beta's allocated ID.
+            title("New item title", "Gamma")
+            tap("Add item")
+            find("Item count: 2")
+            find("Local storage ready")
+            find("item-row-item-003", "resource-id")
+            result["new_create"] = database_state("new-create")
+            assert result["new_create"]["items"][0] == result["before_kill"]["items"][0]
+            assert [item["id"] for item in result["new_create"]["items"]] == ["item-001", "item-003"]
+            assert result["new_create"]["items"][1]["title"] == "Gamma"
+            tap("Delete Gamma")
+            find("Item count: 1")
+            find("Local storage ready")
+            assert_final(snapshot("final"))
+            assert database_state("final")["items"] == result["before_kill"]["items"]
+        assert os.getpid() == result["host_pid"]
+        result["status"] = "BASELINE_LOSS_REPRODUCED" if args.expect_loss else "PASS"
+    except Exception as error:
+        result["status"] = "FAIL"
+        result["error"] = repr(error)
+        raise
+    finally:
+        if network_before is not None:
+            adb("shell", "cmd", "connectivity", "airplane-mode", "enable" if network_before["airplane_mode_on"] == "1" else "disable")
+            adb("shell", "svc", "wifi", "enable" if network_before["wifi_on"] == "1" else "disable")
+            adb("shell", "svc", "data", "enable" if network_before["mobile_data"] == "1" else "disable")
+            result["network_restored"] = wait_for_network(network_state, network_before)
+            if result["network_restored"] != network_before:
+                result["status"] = "FAIL"
+                result["error"] = "Could not restore original network settings"
+        (evidence / "result.json").write_text(json.dumps(result, indent=2))
+        print(json.dumps(result, indent=2), flush=True)
+        if result["status"] == "FAIL":
+            raise AssertionError(result["error"])
+
+
+if __name__ == "__main__":
+    main()
diff --git a/src/App.tsx b/src/App.tsx
index 333e1d1..36ffa7b 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -1,45 +1,92 @@
-import React, {useReducer, useRef, useState} from 'react';
+import React, {useEffect, useRef, useState} from 'react';
 import {Button, Keyboard, Pressable, SafeAreaView, ScrollView, StyleSheet, Text, TextInput, View} from 'react-native';
-import {itemsReducer} from './items';
+import {Item} from './items';
+import {ItemMutation, ItemStore, openItemStore} from './itemStore';
 
-export default function App() {
-  const [items, dispatch] = useReducer(itemsReducer, []);
+export default function App({openStore = openItemStore}: {openStore?: () => Promise<ItemStore>}) {
+  const [items, setItems] = useState<Item[]>([]);
+  const [ready, setReady] = useState(false);
+  const [busy, setBusy] = useState(true);
+  const [error, setError] = useState<string | null>(null);
+  const [openAttempt, setOpenAttempt] = useState(0);
+  const store = useRef<ItemStore | null>(null);
+  const busyRef = useRef(true);
   const [draft, setDraft] = useState('');
   const [editingId, setEditingId] = useState<string | null>(null);
-  const nextId = useRef(1);
 
-  function saveTitle() {
+  useEffect(() => {
+    let active = true;
+    busyRef.current = true;
+    setBusy(true);
+    setError(null);
+    openStore().then(async opened => {
+      const saved = await opened.read();
+      if (active) {
+        store.current = opened;
+        setItems(saved);
+        setReady(true);
+      }
+    }).catch(reason => {
+      if (active) {setError(`Could not open local database: ${String(reason.message ?? reason)}`);}
+    }).finally(() => {
+      if (active) {busyRef.current = false; setBusy(false);}
+    });
+    return () => {active = false;};
+  }, [openStore, openAttempt]);
+
+  async function mutate(action: ItemMutation): Promise<boolean> {
+    if (!store.current || busyRef.current) {return false;}
+    busyRef.current = true;
+    setBusy(true);
+    setError(null);
+    try {
+      setItems(await store.current.mutate(action));
+      return true;
+    } catch (reason) {
+      setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);
+      return false;
+    } finally {
+      busyRef.current = false;
+      setBusy(false);
+    }
+  }
+
+  async function saveTitle() {
     if (!draft.trim()) {
       return;
     }
-    if (editingId !== null) {
-      dispatch({type: 'rename', id: editingId, title: draft, now: Date.now()});
-    } else {
-      const id = `item-${String(nextId.current++).padStart(3, '0')}`;
-      dispatch({type: 'create', id, title: draft, now: Date.now()});
+    const saved = await mutate(editingId !== null
+      ? {type: 'rename', id: editingId, title: draft, now: Date.now()}
+      : {type: 'create', title: draft, now: Date.now()});
+    if (saved) {
+      setEditingId(null);
+      setDraft('');
+      Keyboard.dismiss();
     }
-    setEditingId(null);
-    setDraft('');
-    Keyboard.dismiss();
   }
 
   return (
     <SafeAreaView style={styles.screen}>
       <Text style={styles.heading}>Offline Item Tracker</Text>
-      <Text>Process memory only</Text>
+      <Text accessibilityLabel={busy ? 'Local storage busy' : ready ? (error ? 'Local storage error' : 'Local storage ready') : 'Local storage unavailable'}>
+        {busy ? (ready ? 'Saving locally…' : 'Opening local database…') : ready ? (error ? 'Change not saved' : 'Saved locally') : 'Local database unavailable'}
+      </Text>
+      {error !== null && <Text accessibilityRole="alert">{error}</Text>}
+      {!ready && !busy && <Button title="Retry opening database" onPress={() => setOpenAttempt(value => value + 1)} />}
       <TextInput
         accessibilityLabel={editingId === null ? 'New item title' : 'Edit item title'}
         placeholder="Item title"
         value={draft}
+        editable={ready && !busy}
         onChangeText={setDraft}
         onSubmitEditing={saveTitle}
         style={styles.input}
       />
-      <Button title={editingId === null ? 'Add item' : 'Save title'} accessibilityLabel={editingId === null ? 'Add item' : 'Save title'} onPress={saveTitle} disabled={!draft.trim()} />
-      {editingId !== null && <Button title="Cancel edit" onPress={() => {setEditingId(null); setDraft('');}} />}
-      <Text accessibilityLabel={`Item count: ${items.length}`} style={styles.count}>
+      <Button title={editingId === null ? 'Add item' : 'Save title'} accessibilityLabel={editingId === null ? 'Add item' : 'Save title'} onPress={saveTitle} disabled={!ready || busy || !draft.trim()} />
+      {editingId !== null && <Button title="Cancel edit" disabled={busy} onPress={() => {setEditingId(null); setDraft('');}} />}
+      {ready && <Text accessibilityLabel={`Item count: ${items.length}`} style={styles.count}>
         {items.length} {items.length === 1 ? 'item' : 'items'}
-      </Text>
+      </Text>}
       <ScrollView keyboardShouldPersistTaps="handled">
         {items.map(item => (
           <View key={item.id} testID={`item-row-${item.id}`} style={styles.row}>
@@ -48,15 +95,16 @@ export default function App() {
               accessibilityRole="checkbox"
               accessibilityLabel={`Mark ${item.title} ${item.completed ? 'incomplete' : 'complete'}`}
               accessibilityState={{checked: item.completed}}
-              onPress={() => dispatch({type: 'toggle', id: item.id, now: Date.now()})}
+              disabled={busy}
+              onPress={() => mutate({type: 'toggle', id: item.id, now: Date.now()})}
               style={styles.toggle}>
               <Text>{item.completed ? 'Completed' : 'Incomplete'}</Text>
             </Pressable>
             <View style={styles.actions}>
-              <Button title="Edit" accessibilityLabel={`Edit ${item.title}`} onPress={() => {setEditingId(item.id); setDraft(item.title);}} />
-              <Button title="Delete" accessibilityLabel={`Delete ${item.title}`} onPress={() => {
-                dispatch({type: 'delete', id: item.id});
-                if (editingId === item.id) {setEditingId(null); setDraft('');}
+              <Button title="Edit" accessibilityLabel={`Edit ${item.title}`} disabled={busy} onPress={() => {setEditingId(item.id); setDraft(item.title);}} />
+              <Button title="Delete" accessibilityLabel={`Delete ${item.title}`} disabled={busy} onPress={async () => {
+                const saved = await mutate({type: 'delete', id: item.id});
+                if (saved && editingId === item.id) {setEditingId(null); setDraft('');}
               }} />
             </View>
           </View>
diff --git a/src/itemStore.ts b/src/itemStore.ts
new file mode 100644
index 0000000..f2bd259
--- /dev/null
+++ b/src/itemStore.ts
@@ -0,0 +1,129 @@
+import SQLite, {SQLResultSet, SQLTransaction, WebsqlDatabase} from 'react-native-sqlite-2';
+import {Item, ItemAction, itemsReducer} from './items';
+
+export const DATABASE_NAME = 'items.db';
+export const SCHEMA_VERSION = 1;
+
+export type ItemRow = {
+  id: string;
+  title: string;
+  completed: number;
+  version: number;
+  updated_at: number;
+};
+
+export function itemToRow(item: Item): ItemRow {
+  return {id: item.id, title: item.title, completed: item.completed ? 1 : 0,
+    version: item.version, updated_at: item.updatedAt};
+}
+
+export function rowToItem(row: ItemRow): Item {
+  if (typeof row.id !== 'string' || !row.id || typeof row.title !== 'string' || !row.title.trim()
+      || (row.completed !== 0 && row.completed !== 1)
+      || !Number.isSafeInteger(row.version) || row.version < 0
+      || !Number.isSafeInteger(row.updated_at)) {
+    throw new Error('Invalid Item in the local database');
+  }
+  return {id: row.id, title: row.title, completed: row.completed === 1,
+    version: row.version, updatedAt: row.updated_at};
+}
+
+export type ItemMutation = Exclude<ItemAction, {type: 'create'}>
+  | {type: 'create'; title: string; now: number};
+
+export interface ItemStore {
+  read(): Promise<Item[]>;
+  mutate(action: ItemMutation): Promise<Item[]>;
+}
+
+function readItems(tx: SQLTransaction, callback: (items: Item[]) => void) {
+  tx.executeSql('SELECT id, title, completed, version, updated_at FROM items ORDER BY rowid', [], (_, result: SQLResultSet) => {
+    const items: Item[] = [];
+    for (let i = 0; i < result.rows.length; i++) {
+      items.push(rowToItem(result.rows.item(i)));
+    }
+    callback(items);
+  });
+}
+
+// Resolve only in the transaction's success callback, after native COMMIT.
+// A failed statement/commit rolls back and must not publish its candidate state.
+class SqliteItemStore implements ItemStore {
+  constructor(private readonly database: WebsqlDatabase) {}
+
+  read(): Promise<Item[]> {
+    return new Promise((resolve, reject) => {
+      let items: Item[] = [];
+      this.database.readTransaction(tx => readItems(tx, result => {items = result;}), reject, () => resolve(items));
+    });
+  }
+
+  mutate(action: ItemMutation): Promise<Item[]> {
+    return new Promise((resolve, reject) => {
+      let committed: Item[] = [];
+      this.database.transaction(tx => {
+        readItems(tx, current => {
+          const apply = (identified: ItemAction) => {
+            const next = itemsReducer(current, identified);
+            if (identified.type === 'delete') {
+              tx.executeSql('DELETE FROM items WHERE id = ?', [identified.id]);
+            } else {
+              const item = next.find(value => value.id === identified.id);
+              if (item && item !== current.find(value => value.id === identified.id)) {
+                const row = itemToRow(item);
+                if (identified.type === 'create') {
+                  tx.executeSql('INSERT INTO items (id, title, completed, version, updated_at) VALUES (?, ?, ?, ?, ?)',
+                    [row.id, row.title, row.completed, row.version, row.updated_at]);
+                } else {
+                  tx.executeSql('UPDATE items SET title = ?, completed = ?, version = ?, updated_at = ? WHERE id = ?',
+                    [row.title, row.completed, row.version, row.updated_at, row.id]);
+                }
+              }
+            }
+            readItems(tx, result => {committed = result;});
+          };
+
+          if (action.type === 'create' && action.title.trim()) {
+            tx.executeSql('SELECT next_id FROM local_identity WHERE singleton = 1', [], (_, result) => {
+              const nextId = result.rows.item(0)?.next_id;
+              if (!Number.isSafeInteger(nextId) || nextId < 1) {
+                throw new Error('Invalid local Item identity sequence');
+              }
+              const id = `item-${String(nextId).padStart(3, '0')}`;
+              if (current.some(item => item.id === id)) {
+                throw new Error('Local Item identity already exists');
+              }
+              // The allocator and insert commit together; deletion never rewinds it.
+              tx.executeSql('UPDATE local_identity SET next_id = next_id + 1 WHERE singleton = 1');
+              apply({...action, id});
+            });
+          } else if (action.type !== 'create') {
+            apply(action);
+          } else {
+            committed = current;
+          }
+        });
+      }, reject, () => resolve(committed));
+    });
+  }
+}
+
+export async function openItemStore(name = DATABASE_NAME): Promise<ItemStore> {
+  const database = SQLite.openDatabase(name);
+  await new Promise<void>((resolve, reject) => {
+    database.transaction(tx => {
+      tx.executeSql('PRAGMA user_version', [], (_, result) => {
+        const version = result.rows.item(0).user_version;
+        if (version === 0) {
+          tx.executeSql('CREATE TABLE items (id TEXT PRIMARY KEY NOT NULL, title TEXT NOT NULL CHECK(length(trim(title)) > 0), completed INTEGER NOT NULL CHECK(completed IN (0, 1)), version INTEGER NOT NULL CHECK(version >= 0), updated_at INTEGER NOT NULL)');
+          tx.executeSql('CREATE TABLE local_identity (singleton INTEGER PRIMARY KEY CHECK(singleton = 1), next_id INTEGER NOT NULL CHECK(next_id > 0))');
+          tx.executeSql('INSERT INTO local_identity (singleton, next_id) VALUES (1, 1)');
+          tx.executeSql(`PRAGMA user_version = ${SCHEMA_VERSION}`);
+        } else if (version !== SCHEMA_VERSION) {
+          throw new Error(`Unsupported local database schema ${version}`);
+        }
+      });
+    }, reject, resolve);
+  });
+  return new SqliteItemStore(database);
+}
diff --git a/verification/M02.md b/verification/M02.md
new file mode 100644
index 0000000..8c910fa
--- /dev/null
+++ b/verification/M02.md
@@ -0,0 +1,221 @@
+# M02 verification — attempt 1
+
+- Thread: M02; branch: track/react-native.
+- Spec-Revision: ed7baa246ee947c6778e0f84751c9b91cec7abfe.
+- START: c45b07972e368a45d9bd8c3cd294f3aeea49bf8b (verified M01).
+- Frozen M02 scenario SHA-256: 48a5002b9d340cd4b42157e7fa3604498f3348e5d359436d76f69ca087f0ac6a.
+- Android: the same API 34 Pixel 6 ARM64 device, emulator-5554.
+- Raw evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M02/`.
+
+## New constraint and reproduction
+
+All frozen specification documents, RUN.md, M01/M02 scenario records, TRACK.md,
+M01 verification and the React Native code were read before implementation.
+No other track's implementation was read. No fixed inputs or acceptance changed.
+
+The baseline APK was copied before any production code change. Its SHA-256 is
+`7b93afdcd3e45104d13451549c04dfbb13db8779a76fa302325b0d0b3166b284`, identical to
+the verified M01 APK. The external Python harness performs the exact sequence:
+empty installation; Alpha, Beta, rename first to Alpha edited, true/false/true,
+delete Beta; assert count 1 and first identity `item-001`; only then force-stop.
+
+Baseline invocation 1 (exit 0, expected M01 loss reproduced):
+
+```sh
+python3 scripts/verify_m02.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --apk /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M02/m01-baseline.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M02/baseline-01 --expect-loss
+```
+
+The host harness PID was 65648. App PID **8318** was force-stopped; `pidof`
+confirmed no remaining process (expected exit 1), and cold relaunch produced
+PID **8993**. The restored UI was empty. `baseline-01/result.json`, the full
+`commands.json` argument/output/exit ledger, XML dumps and screenshots before
+and after termination are retained. The device lease was granted before use and
+released afterward. This block made no network or device profile change.
+
+## Verification ledger
+
+Every command below ran from the React Native branch root, except Gradle commands
+which ran from `android/`. npm commands used this exact prefix:
+
+```sh
+env PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH /opt/homebrew/bin/npm
+```
+
+Gradle commands used this prefix (the connected command also set
+`ANDROID_SERIAL=emulator-5554`):
+
+```sh
+env PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home ANDROID_HOME=/opt/homebrew/share/android-commandlinetools GRADLE_USER_HOME=/private/tmp/mobile-systems-evolution-ed7baa2/gradle-react-native
+```
+
+stdout and stderr were redirected together to the named raw log under the evidence
+root. Setup commands, including both compatibility-patch generations:
+
+| Invocation | Command after the npm prefix | Exit | Raw log |
+|---|---|---:|---|
+| Dependency 1 | `install --save-exact react-native-sqlite-2@3.6.3 --ignore-scripts --no-audit --no-fund --cache /private/tmp/mobile-systems-evolution-ed7baa2/npm-cache` | 0 | npm-install-sqlite.log |
+| Dependency 2 | `install --save-dev --save-exact patch-package@8.0.0 --ignore-scripts --no-audit --no-fund --cache /private/tmp/mobile-systems-evolution-ed7baa2/npm-cache` | 0 | npm-install-patch.log |
+| Patch 1 | `env PATH=/Users/woopinbell/.local/share/fnm/node-versions/v22.22.0/installation/bin:$PATH ./node_modules/.bin/patch-package react-native-sqlite-2` (no npm prefix) | 0 | patch-package.log |
+| Patch 2 | same patch-package command | 0 | patch-package-02.log |
+| Postinstall 1 | `run postinstall` | 0 | postinstall-01.log; pinned patch applies successfully |
+
+The native [SQLite library](https://github.com/craftzdog/react-native-sqlite-2)
+provides Android persistence and is autolinked. Its native API handles SQLite;
+there is no application-specific native module. The pinned compatibility patch
+sets AGP's namespace, removes the obsolete manifest namespace, and omits the
+library's unused null NDK version. SQLite's prebuilt Android library is packaged
+without installing an NDK. npm's transitive deprecation warnings remain in the
+setup logs; unrelated dependencies were not upgraded.
+
+| Invocation | Exact command after the applicable prefix | Exit | Result | Raw log |
+|---|---|---:|---|---|
+| Typecheck 1 | `run typecheck` | 2 | Host test fixture lacked Node type declarations for fs/os/path | typecheck-01.log |
+| Jest 1 | `test -- --runInBand --watchman=false` | 0 | 2 suites, 10 tests passed; React Animated warnings in the default iOS host mock | jest-01.log |
+| Typecheck 2 | `run typecheck` | 0 | Host fixture now references installed Node types | typecheck-02.log |
+| Jest 2 | `test -- --runInBand --watchman=false` | 1 | 9 passed, 1 failed: Android uppercases Button text, so the retry test's exact text selector was wrong | jest-02.log |
+| Jest 3 | `test -- --runInBand --watchman=false` | 0 | 2 suites, all 10 tests passed; no React act warnings | jest-03.log |
+| Typecheck 3 | `run typecheck` | 0 | Final host type check | typecheck-03.log |
+| Build 1 | `./gradlew --no-daemon :app:assembleDebug :app:assembleDebugAndroidTest` | 1 | Upstream SQLite Gradle configuration passed a null NDK version | build-01.log |
+| Build 2 | same Gradle build command | 0 | Both APKs built; 99 tasks, 47 executed | build-02.log |
+| Android 1 | `./gradlew --no-daemon :app:connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=com.mse.reactnative.M01ItemUiTest` | 0 | The unchanged M01 test passed on API 34: 1 test, 0 failures/skips | android-01.log |
+
+The host mock now selects Android API 34 controls, matching the actual target.
+The retry selector uses the button's accessible name without assuming visual
+capitalization. Neither change alters any frozen Item mutation or acceptance.
+The only remaining host warning is Node's experimental built-in SQLite notice.
+
+Before Android invocation 1, this explicit fixture reset exited 0 (raw output
+`android-clear-01.log`):
+
+```sh
+/opt/homebrew/share/android-commandlinetools/platform-tools/adb -s emulator-5554 shell pm clear com.mse.reactnative
+```
+
+The original instrumentation APK/test source is unchanged. Android JUnit XML and
+logcat are preserved under `android-results-01/`. This regression uses native
+React Native controls, not the host component renderer. The syntax check
+`python3 -m py_compile scripts/verify_m02.py` ran twice and exited 0 both times,
+including after the final cleanup-only harness adjustment.
+
+## Persistence and error coverage
+
+The existing host suites now test the production SQL and library JavaScript
+transaction callbacks against Node SQLite files. Only the native bridge boundary
+is replaced on the host; no in-memory repository substitute is used. Files are
+closed and reopened, and the exact round-trip fixture is:
+
+```json
+{"id":"item-001","title":"Alpha edited","completed":true,"version":0,"updatedAt":1700000006000}
+```
+
+All five fields and both completion encodings are checked. This explicit M02
+fixture is separate from the unchanged M01 local-edit counter: the fixed mutation
+sequence still produces version 5 and host timestamp 1700000005000. It was not
+rewritten to make the round-trip fixture pass.
+
+Coverage includes the fixed CRUD sequence through SQLite, file reopening, durable
+identity allocation after deletion/restart, rollback of both an INSERT failure
+and a COMMIT failure, preservation of the prior UI/draft on failed writes, opening
+failure with retry, and rejection of a newer schema while leaving its rows and
+schema version intact. UI success and published state follow native COMMIT.
+
+## Actual offline process restart
+
+Restart invocation 1:
+
+```sh
+python3 scripts/verify_m02.py --adb /opt/homebrew/share/android-commandlinetools/platform-tools/adb --serial emulator-5554 --apk android/app/build/outputs/apk/debug/app-debug.apk --evidence /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M02/restart-01
+```
+
+**Exit 1 / FAIL is preserved.** Every application durability/identity assertion
+completed successfully; the subsequent network-cleanup check observed a still
+pending asynchronous mobile-data change and failed. This invocation is not
+relabeled as an overall pass.
+
+The external host harness PID **71735** survived app force-stop. The app changed
+from PID **10570** to **11335**, with no PID present between them. No app-data
+clear, reinstall, or database replacement occurs between force-stop and relaunch.
+The final operation was acknowledged by `Local storage ready` before termination.
+Before and after the new process opened its native SQLite database, the exact
+one-Item row was:
+
+```json
+{"id":"item-001","title":"Alpha edited","completed":true,"version":5,"updatedAt":1787874838153}
+```
+
+The database schema version stayed 1 and the persisted identity allocator stayed
+3. Integrity checks passed. Deleted `item-002` / Beta was absent from both the
+native database and the real Android UI. A separate post-acceptance create of
+Gamma allocated **item-003** without changing the original Item. Deleting that
+probe restored the exact original one-Item state; the allocator advanced to 4.
+Database copies and JSON inspections are `restart-01/before-kill.*`,
+`after-restart.*`, `new-create.*`, and `final.*`, with UI XML/screenshots and every
+adb invocation/output/exit recorded in `commands.json`.
+
+During restart, airplane mode was 1, Wi-Fi 0, and mobile data 0.
+`offline-connectivity.txt` reports `Active default network: none`. The tested APK
+also has no INTERNET permission (`apk-permissions.log`, captured with
+`/opt/homebrew/share/android-commandlinetools/build-tools/35.0.0/aapt2 dump permissions android/app/build/outputs/apk/debug/app-debug.apk`).
+
+Original device settings were 0/1/1. The harness's immediate cleanup check saw
+0/1/0 and therefore failed. A subsequent read-only cleanup confirmation, before
+releasing the lease, found **0/1/1**, exactly the originals. Its three commands were
+`adb -s emulator-5554 shell settings get global` followed respectively by
+`airplane_mode_on`, `wifi_on`, and `mobile_data` (all exit 0); full executable paths,
+arguments and outputs are in `network-cleanup-01.json`. No additional setting
+mutation was needed; the service change finished asynchronously.
+
+The final harness adds only a bounded 10-second wait, with 100-ms intervals, for
+cleanup settings to equal the originals. It does not alter the frozen CRUD order,
+kill point, offline condition, or database assertions. This wait was checked on
+the host with a controlled clock and state reader: delayed restoration succeeds,
+and expiration preserves the mismatch rather than claiming success. Invocation:
+
+```sh
+python3 /private/tmp/mobile-systems-evolution-ed7baa2/evidence/react-native/M02/verify-cleanup-wait.py
+```
+
+Exit 0; two checks passed (`cleanup-wait-check-01.log`). **The final cleanup-wait
+change has not had another full device harness run in this attempt.** Main must
+run the final committed harness independently before progress tagging.
+
+## Harness provenance and artifacts
+
+The baseline source was not snapshotted or hashed at execution time. Its exact
+initial patch was reconstructed afterward by reversing only the later additions,
+and is retained as `verify_m02-baseline-reconstructed.py`. The hash is explicitly
+retrospective; baseline APK, command/PID ledger and UI evidence were captured at
+execution time. From that source to restart invocation 1, the fixed UI sequence
+and process termination were retained; additions were native database inspection,
+the explicit saved-state wait, temporary offline setup/restoration, and the
+separate new-ID probe. From restart invocation 1 to final, only the cleanup wait
+changed.
+
+| Artifact | SHA-256 |
+|---|---|
+| Reconstructed baseline harness | `752e757c57f3e9c9b7e0aeba9b4be60352977b1dae418a6ad6632b9e9d8cb80d` |
+| Harness captured before restart invocation 1 (`verify_m02.py` in evidence) | `f083a6b43cbf57d225790391472cbe7867625ad86b12fbfe641b76ac908cc4e6` |
+| Final repository harness (`verify_m02-final.py` in evidence) | `2ab0f3d2ca44c4e11049e7d19f50b37837df8a7f27aa33b41e3352ae304c2ca4` |
+| M02 app-debug.apk | `05527e919b30b1e1d9de33e21d3c55c1720dd87d83eb977c2d553f8108126808` |
+| Unchanged M01 Android test APK | `7259ba83ec046d34a3e489ebec46ddcf9ce89dd136a934784c0c89669795b7c1` |
+
+Invocation totals: 3 TypeScript checks (1 failed), 3 Jest runs (one run had one
+failed selector; final run 10/10 passed), 2 APK builds (1 configuration failure),
+1 Android instrumentation run (1/1 passed), 1 M01 baseline process-loss run
+(expected loss reproduced), 1 M02 restart run (application assertions completed;
+cleanup failure retained), 1 read-only cleanup confirmation, 2 Python syntax
+checks, and 1 controlled cleanup-wait check invocation with 2 passing cases.
+The baseline ledger contains 81 adb commands; the M02 restart ledger contains 136.
+Each has one expected exit-1 `pidof` after force-stop, proving the process is gone.
+There were exactly 3 real UI sequence executions across baseline, unchanged M01
+regression, and M02 restart. No other device/API, performance benchmark, battery
+soak, or scenario tuning was used. Both emulator lease blocks were explicitly
+released, the second only after actual connectivity restoration was confirmed.
+
+## Handoff boundary
+
+Production behavior, host tests, APK build, real UI CRUD, actual process death,
+native field durability and new-ID allocation have been exercised as described.
+No product defect remains known. The final cleanup-only harness wait awaits the
+main agent's independent full device rerun; the failed first run remains evidence.
+This subagent creates no tag and starts no later Thread.


