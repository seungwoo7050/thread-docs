## `feat(M08): retain editor state across Activity remounts`

diff --git a/TRACK.md b/TRACK.md
index 04618e6..9d44513 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -169,6 +169,28 @@ identity errors retain their existing semantics. No migration framework is added
 See `verification/M07.md` for the preserved baseline failure/repair, host checks,
 and main's official Android result.
 
+## M08: editor restoration within the application runtime (phase-1)
+
+The selected Item ID and unsubmitted draft now have a separate, small in-memory
+owner for the current JavaScript runtime. Each remounted screen initializes its
+editor from that snapshot; normal edit/cancel/successful-save handlers replace it
+synchronously. Activity recreation keeps this runtime, as measured on Android.
+Process or runtime termination starts with an empty editor: no durable draft
+recovery is promised, and no draft is written to SQLite or the upload queue.
+
+The component owns its mounted flag and opening-effect disposal guard. Handlers
+from an unmounted root cannot start mutations/synchronization or alter the new
+editor. A late Save completion cannot clear its draft or dismiss its keyboard;
+failed saves retain the draft. Already-started durable work still belongs to the
+existing persistence/sync implementation. No event subscription, native hook,
+schema change, state framework or dependency is introduced.
+
+The frozen native test performs one CREATED→RESUMED cycle and one actual
+ActivityScenario recreation, recording PID/Application/Activity identities,
+rendered controls and native database copies. Host tests separately cover editor
+ownership and disposed callbacks. See `verification/M08.md` for the unchanged-app
+baseline, raw commands and main's final Android verification status.
+
 ## Toolchain and commands
 
 Use Node 22.22.0, npm, JDK 17, Android SDK platform 35/build-tools 35.0.0, and the fixed
diff --git a/__tests__/App.test.tsx b/__tests__/App.test.tsx
index ef20c47..183ab5d 100644
--- a/__tests__/App.test.tsx
+++ b/__tests__/App.test.tsx
@@ -1,11 +1,16 @@
 import React from 'react';
+import {Button, Keyboard} from 'react-native';
 import {act, fireEvent, render, screen, waitFor} from '@testing-library/react-native';
-import App from '../src/App';
+import RootApp, {createEditorMemory} from '../src/App';
 import {openItemStore} from '../src/itemStore';
 import {closeDatabases, failNextSql} from './sqliteNative';
 import {ForegroundSync, JsonRequest} from '../src/sync';
 
 const saved = () => waitFor(() => expect(screen.getByLabelText('Local storage ready')).toBeTruthy());
+let editorMemory: ReturnType<typeof createEditorMemory>;
+beforeEach(() => {editorMemory = createEditorMemory();});
+// Each test owns one JS-session editor; remounts within it retain that owner.
+const App = (props: React.ComponentProps<typeof RootApp>) => <RootApp editorMemory={editorMemory} {...props} />;
 
 test('M01 fixed sequence maps stable Item identity to the rendered list', async () => {
   let clock = 1700000000000;
@@ -393,3 +398,159 @@ test('M07 visible canonical-wins notice survives reopening and a fresh explicit
   expect(await reopened.read()).toEqual([input.explicitItem]);
   expect(replies).toEqual([]);
 });
+
+const m08Input = require('../android/app/src/androidTest/assets/m08-inputs.json');
+
+test('M08 the selected Item and unsubmitted draft survive root remount without a durable mutation', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  const mutate = jest.spyOn(store, 'mutate');
+  const synchronize = jest.fn(async () => {});
+  const props = {openStore: async () => store, createSync: () => ({initialized: true, identityPrefix: 'device', synchronize})};
+  const first = render(<App {...props} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), m08Input.draft);
+  expect(editorMemory.current).toEqual({editingId: 'ui-001', draft: m08Input.draft});
+  expect(await store.read()).toEqual([m08Input.seed]);
+  expect(await store.readPending()).toEqual([]);
+  first.unmount();
+  render(<App {...props} />);
+  await saved();
+  expect(screen.getByLabelText('Edit item title').props.value).toBe(m08Input.draft);
+  expect(screen.getByLabelText('Save title')).toBeTruthy();
+  expect(screen.getByTestId('item-title-ui-001').props.children).toBe('Saved title');
+  expect(await store.readPending()).toEqual([]);
+  expect(mutate).not.toHaveBeenCalled();
+  expect(synchronize).not.toHaveBeenCalled();
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await saved();
+  expect(mutate).toHaveBeenCalledTimes(1);
+  expect(await store.read()).toEqual([{...m08Input.seed, title: m08Input.draft, version: 2, updatedAt: expect.any(Number)}]);
+  expect(await store.readPending()).toEqual([expect.objectContaining({kind: 'rename', itemId: 'ui-001',
+    payload: {title: m08Input.draft, baseVersion: 1}, dispatched: false})]);
+  expect(editorMemory.current).toEqual({editingId: null, draft: ''});
+  expect(screen.getByLabelText('New item title').props.value).toBe('');
+});
+
+test('M08 callbacks retained from an unmounted editor cannot edit, submit or synchronize the new root', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  const mutate = jest.spyOn(store, 'mutate');
+  const synchronize = jest.fn(async () => {});
+  const props = {openStore: async () => store, createSync: () => ({initialized: true, identityPrefix: 'device', synchronize})};
+  const first = render(<App {...props} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), m08Input.draft);
+  const oldChange = screen.getByLabelText('Edit item title').props.onChangeText;
+  const oldSubmit = screen.getByLabelText('Edit item title').props.onSubmitEditing;
+  const oldSync = screen.UNSAFE_getAllByType(Button).find(button => button.props.title === 'Synchronize')!.props.onPress;
+  first.unmount();
+  render(<App {...props} />);
+  await saved();
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), 'Current root draft');
+  await act(async () => {
+    oldChange('Late old callback');
+    await oldSubmit();
+    await oldSync();
+  });
+  expect(editorMemory.current).toEqual({editingId: 'ui-001', draft: 'Current root draft'});
+  expect(screen.getByLabelText('Edit item title').props.value).toBe('Current root draft');
+  expect(mutate).not.toHaveBeenCalled();
+  expect(synchronize).not.toHaveBeenCalled();
+  expect(await store.read()).toEqual([m08Input.seed]);
+  expect(await store.readPending()).toEqual([]);
+});
+
+test('M08 a late Save completion cannot clear the remounted draft or dismiss its keyboard', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  let release!: () => void;
+  let committed!: () => void;
+  const response = new Promise<void>(resolve => {release = resolve;});
+  const didCommit = new Promise<void>(resolve => {committed = resolve;});
+  const originalMutate = store.mutate.bind(store);
+  const mutate = jest.spyOn(store, 'mutate').mockImplementation(async (...args) => {
+    const rows = await originalMutate(...args);
+    committed();
+    await response;
+    return rows;
+  });
+  const dismiss = jest.spyOn(Keyboard, 'dismiss');
+  const pendingReads = jest.spyOn(store, 'readPending');
+  const props = {openStore: async () => store};
+  try {
+    const first = render(<App {...props} />);
+    await saved();
+    fireEvent.press(screen.getByLabelText('Edit Saved title'));
+    fireEvent.changeText(screen.getByLabelText('Edit item title'), m08Input.draft);
+    fireEvent.press(screen.getByLabelText('Save title'));
+    await act(async () => {await didCommit;});
+    first.unmount();
+    render(<App {...props} />);
+    await saved();
+    fireEvent.changeText(screen.getByLabelText('Edit item title'), 'New draft after remount');
+    const readsBeforeCompletion = pendingReads.mock.calls.length;
+    await act(async () => {release(); await response;});
+    expect(editorMemory.current).toEqual({editingId: 'ui-001', draft: 'New draft after remount'});
+    expect(screen.getByLabelText('Edit item title').props.value).toBe('New draft after remount');
+    expect(dismiss).not.toHaveBeenCalled();
+    expect(pendingReads).toHaveBeenCalledTimes(readsBeforeCompletion);
+    expect(mutate).toHaveBeenCalledTimes(1);
+    expect(await store.readPending()).toHaveLength(1);
+  } finally {
+    await act(async () => {release(); await response;});
+    dismiss.mockRestore();
+    mutate.mockRestore();
+  }
+});
+
+test('M08 failed submission keeps the remountable draft; explicit cancel clears only editor memory', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  const props = {openStore: async () => store};
+  const first = render(<App {...props} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), m08Input.draft);
+  failNextSql(/^INSERT INTO pending_mutations/);
+  fireEvent.press(screen.getByLabelText('Save title'));
+  await waitFor(() => expect(screen.getByLabelText('Local storage error')).toBeTruthy());
+  expect(editorMemory.current).toEqual({editingId: 'ui-001', draft: m08Input.draft});
+  first.unmount();
+  const second = render(<App {...props} />);
+  await saved();
+  expect(screen.getByLabelText('Edit item title').props.value).toBe(m08Input.draft);
+  expect(await store.read()).toEqual([m08Input.seed]);
+  expect(await store.readPending()).toEqual([]);
+  fireEvent.press(screen.getByRole('button', {name: /cancel edit/i}));
+  second.unmount();
+  render(<App {...props} />);
+  await saved();
+  expect(screen.getByLabelText('New item title').props.value).toBe('');
+  expect(screen.queryByLabelText('Save title')).toBeNull();
+  expect(await store.read()).toEqual([m08Input.seed]);
+  expect(await store.readPending()).toEqual([]);
+});
+
+test('M08 a disposed opening effect cannot install a duplicate session or publish into the remounted root', async () => {
+  const store = await openItemStore();
+  await store.replaceSnapshot([m08Input.seed]);
+  let finishOldOpen!: (opened: typeof store) => void;
+  const oldOpening = new Promise<typeof store>(resolve => {finishOldOpen = resolve;});
+  const oldSync = jest.fn(() => ({initialized: true, identityPrefix: 'old', synchronize: jest.fn(async () => {})}));
+  const newSync = jest.fn(() => ({initialized: true, identityPrefix: 'new', synchronize: jest.fn(async () => {})}));
+  const first = render(<App openStore={() => oldOpening} createSync={oldSync} />);
+  first.unmount();
+  render(<App openStore={async () => store} createSync={newSync} />);
+  await saved();
+  fireEvent.press(screen.getByLabelText('Edit Saved title'));
+  fireEvent.changeText(screen.getByLabelText('Edit item title'), m08Input.draft);
+  await act(async () => {finishOldOpen(store); await oldOpening;});
+  expect(oldSync).not.toHaveBeenCalled();
+  expect(newSync).toHaveBeenCalledTimes(1);
+  expect(screen.getByLabelText('Edit item title').props.value).toBe(m08Input.draft);
+  expect(await store.read()).toEqual([m08Input.seed]);
+  expect(await store.readPending()).toEqual([]);
+});
diff --git a/src/App.tsx b/src/App.tsx
index 1ceb0d3..d8c18ff 100644
--- a/src/App.tsx
+++ b/src/App.tsx
@@ -14,12 +14,20 @@ const defaultSync = (store: ItemStore, identityPrefix?: string, testRefreshClock
 
 type RefreshState = {status: 'stale' | 'refreshing' | 'fresh'} | {status: 'error'; message: string};
 
-export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix, testMutationIdentity, testRefreshClock = false}: {
+type EditorState = {editingId: string | null; draft: string};
+export const createEditorMemory = (): {current: EditorState} => ({current: {editingId: null, draft: ''}});
+// Activity recreation remounts this screen while its JS runtime survives. Only
+// editor fields belong here: Items/intents stay in SQLite. A new runtime starts
+// empty; this promises no draft recovery after process death. No subscriptions.
+const applicationEditorMemory = createEditorMemory();
+
+export default function App({openStore = openItemStore, createSync = defaultSync, testIdentityPrefix, testMutationIdentity, testRefreshClock = false, editorMemory = applicationEditorMemory}: {
   openStore?: () => Promise<ItemStore>;
   createSync?: (store: ItemStore, identityPrefix?: string, testRefreshClock?: boolean) => SyncSession;
   testIdentityPrefix?: string;
   testMutationIdentity?: string;
   testRefreshClock?: boolean;
+  editorMemory?: ReturnType<typeof createEditorMemory>;
 }) {
   const [items, setItems] = useState<Item[]>([]);
   const [ready, setReady] = useState(false);
@@ -34,8 +42,20 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   const store = useRef<ItemStore | null>(null);
   const sync = useRef<SyncSession | null>(null);
   const busyRef = useRef(true);
-  const [draft, setDraft] = useState('');
-  const [editingId, setEditingId] = useState<string | null>(null);
+  const mounted = useRef(false);
+  const [{draft, editingId}, setEditor] = useState(() => editorMemory.current);
+
+  useEffect(() => {
+    mounted.current = true;
+    return () => {mounted.current = false;};
+  }, []);
+
+  function updateEditor(next: EditorState) {
+    // A queued handler or old Save completion must not overwrite the new root.
+    if (!mounted.current) {return;}
+    editorMemory.current = next;
+    setEditor(next);
+  }
 
   useEffect(() => {
     let active = true;
@@ -70,57 +90,71 @@ export default function App({openStore = openItemStore, createSync = defaultSync
   }, [openStore, openAttempt, createSync, testIdentityPrefix, testMutationIdentity, testRefreshClock]);
 
   async function reloadPending() {
+    if (!mounted.current) {return;}
     try {
       const pending = await store.current!.readPending();
+      const conflicts = await store.current!.readConflicts();
+      if (!mounted.current) {return;}
       setPendingCount(pending.length);
       setIdentityBlocked(pending.some(operation => operation.terminalError === 'identity_conflict'));
-      setConflictCount((await store.current!.readConflicts()).length);
+      setConflictCount(conflicts.length);
+    }
+    catch {
+      // A failed status read must not claim a committed edit was unsaved.
+      if (mounted.current) {setPendingCount(null); setConflictCount(null);}
     }
-    catch {setPendingCount(null); setConflictCount(null);} // A failed status read must not claim a committed edit was unsaved.
   }
 
   async function mutate(action: ItemMutation): Promise<boolean> {
-    if (!store.current || busyRef.current) {return false;}
+    if (!mounted.current || !store.current || busyRef.current) {return false;}
     busyRef.current = true;
     setBusy(true);
     setError(null);
     try {
       // Every new Item can now upload, even before the first successful refresh.
       // Use the existing distinct namespace immediately; full IDs persist in SQL.
-      setItems(await store.current.mutate(action, sync.current?.identityPrefix));
-      setRefresh({status: 'stale'});
+      const saved = await store.current.mutate(action, sync.current?.identityPrefix);
+      if (mounted.current) {setItems(saved); setRefresh({status: 'stale'});}
       return true;
     } catch (reason) {
-      setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);
+      if (mounted.current) {setError(`Could not save changes: ${reason instanceof Error ? reason.message : String(reason)}`);}
       return false;
     } finally {
       await reloadPending();
       busyRef.current = false;
-      setBusy(false);
+      if (mounted.current) {setBusy(false);}
     }
   }
 
   async function synchronize() {
-    if (!store.current || !sync.current || busyRef.current) {return;}
+    if (!mounted.current || !store.current || !sync.current || busyRef.current) {return;}
     busyRef.current = true;
     setBusy(true);
     setRefresh({status: 'refreshing'});
     try {
       await sync.current.synchronize();
+      if (!mounted.current) {return;}
       const saved = await store.current.read();
       const lastRefresh = await store.current.readLastSuccessfulRefresh();
-      setItems(saved);
-      setLastSuccessfulRefreshAt(lastRefresh);
-      setRefresh({status: 'fresh'});
+      if (mounted.current) {
+        setItems(saved);
+        setLastSuccessfulRefreshAt(lastRefresh);
+        setRefresh({status: 'fresh'});
+      }
     } catch (reason) {
       // A conflict can commit its canonical winner before a later GET fails.
       // Show that committed state, while retaining the refresh error/time.
-      try {setItems(await store.current.read());} catch { /* Keep the last confirmed list. */ }
-      setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});
+      if (mounted.current) {
+        try {
+          const saved = await store.current.read();
+          if (mounted.current) {setItems(saved);}
+        } catch { /* Keep the last confirmed list. */ }
+        if (mounted.current) {setRefresh({status: 'error', message: `Could not refresh: ${reason instanceof Error ? reason.message : String(reason)}`});}
+      }
     } finally {
       await reloadPending();
       busyRef.current = false;
-      setBusy(false);
+      if (mounted.current) {setBusy(false);}
     }
   }
 
@@ -131,9 +165,8 @@ export default function App({openStore = openItemStore, createSync = defaultSync
     const saved = await mutate(editingId !== null
       ? {type: 'rename', id: editingId, title: draft, now: Date.now()}
       : {type: 'create', title: draft, now: Date.now()});
-    if (saved) {
-      setEditingId(null);
-      setDraft('');
+    if (saved && mounted.current) {
+      updateEditor({editingId: null, draft: ''});
       Keyboard.dismiss();
     }
   }
@@ -172,12 +205,12 @@ export default function App({openStore = openItemStore, createSync = defaultSync
         placeholder="Item title"
         value={draft}
         editable={ready && !busy}
-        onChangeText={setDraft}
+        onChangeText={value => updateEditor({editingId, draft: value})}
         onSubmitEditing={saveTitle}
         style={styles.input}
       />
       <Button title={editingId === null ? 'Add item' : 'Save title'} accessibilityLabel={editingId === null ? 'Add item' : 'Save title'} onPress={saveTitle} disabled={!ready || busy || !draft.trim()} />
-      {editingId !== null && <Button title="Cancel edit" disabled={busy} onPress={() => {setEditingId(null); setDraft('');}} />}
+      {editingId !== null && <Button title="Cancel edit" disabled={busy} onPress={() => updateEditor({editingId: null, draft: ''})} />}
       {ready && <Text accessibilityLabel={`Item count: ${items.length}`} style={styles.count}>
         {items.length} {items.length === 1 ? 'item' : 'items'}
       </Text>}
@@ -195,10 +228,10 @@ export default function App({openStore = openItemStore, createSync = defaultSync
               <Text>{item.completed ? 'Completed' : 'Incomplete'}</Text>
             </Pressable>
             <View style={styles.actions}>
-              <Button title="Edit" accessibilityLabel={`Edit ${item.title}`} disabled={busy} onPress={() => {setEditingId(item.id); setDraft(item.title);}} />
+              <Button title="Edit" accessibilityLabel={`Edit ${item.title}`} disabled={busy} onPress={() => updateEditor({editingId: item.id, draft: item.title})} />
               <Button title="Delete" accessibilityLabel={`Delete ${item.title}`} disabled={busy} onPress={async () => {
                 const saved = await mutate({type: 'delete', id: item.id});
-                if (saved && editingId === item.id) {setEditingId(null); setDraft('');}
+                if (saved && editingId === item.id) {updateEditor({editingId: null, draft: ''});}
               }} />
             </View>
           </View>
diff --git a/verification/M08.md b/verification/M08.md
index a86afaa..011c8dd 100644
--- a/verification/M08.md
+++ b/verification/M08.md
@@ -2,7 +2,7 @@
 
 - Spec revision: `61280dd86ce88b6e431f408241c0998a275960aa`.
 - START: verified M07 `34e03d3123e513e0dcd0ea2e55be981f6577249b`.
-- Attempt1; repairs0/2. Current checkpoint: baseline reproduced; product/final verification pending.
+- Attempt1; repairs0/2. Current checkpoint: baseline accepted, host checks PASS; final Android verification pending.
 - Evidence root: `/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/`.
 
 ## Frozen unchanged-app baseline
@@ -12,3 +12,11 @@
 The single [baseline invocation](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01.command.json) exited0 in 17.057s, with 34 adb commands: **BASELINE_LIMIT_REPRODUCED**, not Android acceptance PASS. Native JUnit failed only `M08_RECREATION_DRAFT_OR_SELECTION_LOST`. The draft survived one CREATED→RESUMED cycle; real recreation destroyed Activity46567430 and created232223380 with saved state, losing the editor/draft. PID15260, Application88250467 and ReactContext136876754 stayed identical. All five native databases retained exact `ui-001` / `Saved title` / version1 and pending0; no submit or HTTP request occurred.
 
 [Raw result](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/baseline-android-01/result.json), native archive/DB/UI/lifecycle logs and [main's independent baseline audit](/Users/woopinbell/Desktop/working/workflow/mobile-systems-evolution/threads/evidence/phase-1/react-native/M08/main-baseline-audit.json) retain the complete proof. Cleanup left the app absent, network0/1/1 active and fixture90263 exited0/absent. The denied initial host `ps` probe and subsequent approved absence check are preserved beside the result; neither reran the Android scenario. Production implementation and main's final M08/native CRUD remain pending.
+
+## Implementation and host checks
+
+Following main's baseline acceptance, only `src/App.tsx` changes production behavior: one runtime editor snapshot, remount initialization and component-disposal guards. A late handler/Save completion cannot alter the new editor; a failed Save keeps its draft. No SQL, synchronization algorithm, native source/listener, fixture or dependency changes. The native test, input, harness and instrumentation APK remain exactly frozen.
+
+`host-typecheck-01` **PASS** (1.368s); `host-jest-01` **80/80 PASS** (5.929s), preserving all 75 prior tests. Five focused cases cover draft versus committed state, remount/one explicit mutation, stale handlers, late Save/keyboard cleanup, failed Save/cancel and disposed opening effects. Exact argv, source snapshots and outputs are under the evidence root. No owner fixed Android invocation or repeated baseline occurred; main's final M08/native CRUD is **NOT_RUN** at this checkpoint.
+
+`candidate-app-build-01` **PASS** (10.518s), app-only. Preserved candidate SHA256: `4dc8e86b32b1ca9f72fdf578a57fc197f7c1e81416725111c03e30278b98bbc6`. The only changed APK ZIP entry from M07 is `assets/index.android.bundle`; the instrumentation APK remains exactly frozen. `candidate-preservation-01` confirms host/build source identity and unchanged storage, sync, native implementation and execution support. [Candidate manifest](/private/tmp/mobile-systems-evolution-ed7baa2/evidence/phase-1/react-native/M08/candidate-manifest.json) freezes the final files/APKs and exact main command (same harness without `--baseline`, output `main-android-m08-01`). Main alone runs the required final M08 and native CRUD checks; no other long Android scenarios are repeated for unchanged persistence/sync code.


