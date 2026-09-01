# M03 — Remote Synchronization Baseline

## `test: reproduce isolated local SQLite instances`

diff --git a/__tests__/items.test.ts b/__tests__/items.test.ts
index ce85aad..9a5a708 100644
--- a/__tests__/items.test.ts
+++ b/__tests__/items.test.ts
@@ -88,3 +88,19 @@ test('M02 unsupported schema is rejected without recreating or deleting existing
   expect(connection().prepare('SELECT * FROM items').all()).toEqual(saved.map(itemToRow));
   expect(connection().prepare('PRAGMA user_version').get()?.user_version).toBe(2);
 });
+
+test('M03 baseline: separate local databases cannot observe another instance without synchronization', async () => {
+  const first = await openItemStore('m03-first.db');
+  const second = await openItemStore('m03-second.db');
+  expect(await first.read()).toEqual([]);
+  expect(await second.read()).toEqual([]);
+  const created = await first.mutate({type: 'create', title: 'Gamma', now: 1700000100000});
+  expect(created).toEqual([{id: 'item-001', title: 'Gamma', completed: false, version: 1, updatedAt: 1700000100000}]);
+  expect(await second.read()).toEqual([]);
+  closeDatabases();
+  const reopenedFirst = await (await openItemStore('m03-first.db')).read();
+  const reopenedSecond = await (await openItemStore('m03-second.db')).read();
+  expect(reopenedFirst).toEqual(created);
+  expect(reopenedSecond).toEqual([]);
+  console.info('M03_ISOLATION_BASELINE', JSON.stringify({first: reopenedFirst, second: reopenedSecond}));
+});


