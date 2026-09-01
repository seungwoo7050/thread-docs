# E06 Server-State Cache와 Mutation Invalidation

## `test(state): freeze the held-create duplicate submission`

diff --git a/evidence/E06/baseline.json b/evidence/E06/baseline.json
new file mode 100644
index 0000000..d31addf
--- /dev/null
+++ b/evidence/E06/baseline.json
@@ -0,0 +1,17 @@
+{
+  "mode": "unchanged-START baseline",
+  "start": "ef470a301358932a77457d714c79494e631a8f96",
+  "fixtureSha256": "62c32bb19c2f4cf1f8c958d72014595d11220763d3c4e5f450019c5c64420556",
+  "completed": [
+    "B create/edit/pause/resume/delete and list/detail reload",
+    "A one real Check, latest/history toggle and reload"
+  ],
+  "counterexample": {
+    "submitEvents": 2,
+    "createPosts": 2,
+    "durableRows": 2,
+    "requiredMaximum": 1,
+    "firstRealResponseHeldAfterCommit": true
+  },
+  "heldRoutesSettled": true
+}
diff --git a/evidence/E06/fixtures.md b/evidence/E06/fixtures.md
new file mode 100644
index 0000000..99505b5
--- /dev/null
+++ b/evidence/E06/fixtures.md
@@ -0,0 +1,66 @@
+# E06 frozen server-state scenario
+
+Frozen before E06 test execution or product edits.
+START: `ef470a301358932a77457d714c79494e631a8f96`.
+Spec: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Branch: `track/industry-spring`; attempt1.
+
+## Dataset and fixed sequence
+
+Use the existing isolated browser/API/fixture helpers: PostgreSQL15432,
+`e04_browser` disposable schema, fixture4321, API4322 and UI4323. Refuse occupied
+servers. Authenticate Alice using runtime-only credentials and existing session
+CSRF/Origin setup. Alice initially owns exactly Monitor A, URL
+`http://127.0.0.1:4321/ok`, interval60, enabled true, no CheckRuns.
+
+1. Create Monitor B with `/ok`, interval60 and enabled true through the UI.
+   Inspect its visible article/detail and the existing detail API.
+2. Edit B to `Updated B`, interval90. Inspect list/article detail, open/close
+   existing history, then reload and verify the same canonical values.
+3. Pause B, resume B, delete B, reload the list and verify B's detail is404.
+4. Complete exactly one manual Check for A. Inspect its latest result and
+   single-row history, then hide/reopen history and reload to verify consistency.
+5. Open A's existing edit form. Fill the create form with `Pending C`, `/ok`,
+   interval60, enabled true. Dispatch the first native bubbling, cancelable form
+   submit event. Intercept only this create POST, forward it to the real backend,
+   and hold its real201 response after commit at an explicit promise barrier.
+6. After that committed-response barrier, dispatch exactly one more native form
+   submit event with unchanged form inputs. Both submissions precede release.
+   There must be at most one create POST and one durable `Pending C` row.
+7. While the first response remains held, submit A's edit with name `Rejected A`,
+   interval90, the same URL and enabled true. Intercept exactly this one PUT and
+   return500 with `{"error":{"code":"INTERNAL_ERROR","message":"Fixed E06 failure"}}`.
+   Keep A's authoritative name/interval/latest/history unchanged and show failure.
+8. Still holding the create response, inject the identical500 body for exactly
+   one DELETE of A. A remains visible and authoritative; failure remains visible.
+   Neither failure may clear the unrelated create-pending state.
+9. Release the held real201 response. Create pending clears and exactly one
+   canonical `Pending C` appears. Reload and verify one durable row, A unchanged,
+   and A's original CheckRun retained. Neither injected500 reaches the backend.
+
+The existing UI's article/history is its detail view. No new routes, pagination
+or server-rendering work are part of this scenario.
+
+## Unchanged-START baseline
+
+Run the sequence once at unchanged START, choosing duplicate submission as the
+decisive known gap. After the held first commit and second submit, observe the
+second real create commit if present and record request/row counts against the
+required maximum1. Stop reproduction at that decisive failure; do not execute
+the later500 probes after it. Release the first response during cleanup and await
+owned routes/processes. Do not rerun the baseline or alter inputs/thresholds.
+
+## Execution boundaries
+
+- The hold is an explicit release barrier, not a sleep or chosen latency.
+- Exactly two create form-submit events, one injected update500 and one injected
+  delete500 in post-change verification; no request loops, load or timing sweeps.
+- Existing30-second browser test timeout and default readiness limits remain.
+- Keep the real request path, payload, CSRF and Origin behavior; never fake create
+  success or substitute an illustrative row for the durable database result.
+- No traces, screenshots, videos, storage-state files or credential-bearing logs.
+- Preserve all earlier product fixtures, assertions, evidence and migrations.
+- Reuse independently verified E05 backend results only after confirming backend,
+  dependency and configuration bytes unchanged. Record every new invocation.
+- Zero automatic retries, load runs or parameter sweeps. Stop after required
+  authoring gates pass; root performs independent final acceptance.
diff --git a/tests/browser/server-state.spec.ts b/tests/browser/server-state.spec.ts
new file mode 100644
index 0000000..ea69fbd
--- /dev/null
+++ b/tests/browser/server-state.spec.ts
@@ -0,0 +1,205 @@
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdirSync, readFileSync, writeFileSync } from 'node:fs';
+import type { Page, Route } from '@playwright/test';
+import { test, expect, csrfHeaders, safeRequest } from './authenticated';
+
+const start = 'ef470a301358932a77457d714c79494e631a8f96';
+const baseline = process.env.E06_BASELINE === '1';
+const fixtureUrl = 'http://127.0.0.1:4321/ok';
+
+async function fillCreate(page: Page, name: string) {
+  await page.getByLabel('Name', { exact: true }).fill(name);
+  await page.getByLabel('URL', { exact: true }).fill(fixtureUrl);
+  await page.getByLabel('Interval (seconds)', { exact: true }).fill('60');
+  await page.getByLabel('Enabled', { exact: true }).check();
+}
+
+async function uiResponse(page: Page, path: string, method: string, action: () => Promise<unknown>) {
+  const [response] = await Promise.all([
+    page.waitForResponse(response => new URL(response.url()).pathname === path && response.request().method() === method),
+    action(),
+  ]);
+  return response;
+}
+
+test('fixed server-state lifecycle, held create double-submit and independent mutation failures', async ({ page }) => {
+  if (baseline) expect(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim()).toBe(start);
+  const evidence: Record<string, unknown> = {
+    mode: baseline ? 'unchanged-START baseline' : 'post-change regression', start,
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/E06/fixtures.md')).digest('hex'),
+    completed: [] as string[],
+  };
+  const completed = evidence.completed as string[];
+  const existing = await safeRequest(() => page.request.get('/api/monitors'));
+  expect(existing.status()).toBe(200);
+  for (const row of (await existing.json()).data) {
+    const headers = await csrfHeaders(page.request);
+    expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
+  }
+  const headers = await csrfHeaders(page.request);
+  const seeded = await safeRequest(() => page.request.post('/api/monitors', { headers,
+    data: { name: 'Monitor A', url: fixtureUrl, interval: 60, enabled: true } }));
+  expect(seeded.status()).toBe(201);
+  const aId = (await seeded.json()).data.monitor.id as string;
+  const aPath = `/api/monitors/${aId}`;
+  const authoritative = async (path: string) => {
+    const response = await safeRequest(() => page.request.get(path));
+    return { status: response.status(), body: await response.json() };
+  };
+  const reload = async () => {
+    expect((await uiResponse(page, '/api/monitors', 'GET', () => page.reload())).status()).toBe(200);
+  };
+  const firstCommitted = Promise.withResolvers<void>();
+  const secondCommitted = Promise.withResolvers<void>();
+  const release = Promise.withResolvers<void>();
+  const routes: Promise<void>[] = [];
+  let createPosts = 0;
+  let released = false;
+  let intercepted: ((route: Route) => Promise<void>) | undefined;
+  try {
+    expect((await uiResponse(page, '/api/monitors', 'GET', () => page.goto('/monitors'))).status()).toBe(200);
+    await fillCreate(page, 'Monitor B');
+    const createdB = await uiResponse(page, '/api/monitors', 'POST', () => page.getByRole('button', { name: 'Create monitor', exact: true }).click());
+    expect(createdB.status()).toBe(201);
+    const bId = (await createdB.json()).data.monitor.id as string;
+    const bPath = `/api/monitors/${bId}`;
+    let b = page.getByRole('article', { name: 'Monitor B', exact: true });
+    await expect(b).toBeVisible();
+    expect((await authoritative(bPath)).body.data.monitor).toMatchObject({ name: 'Monitor B', interval: 60 });
+    await b.getByRole('button', { name: 'Edit', exact: true }).click();
+    await b.getByLabel('Edit name', { exact: true }).fill('Updated B');
+    await b.getByLabel('Edit interval (seconds)', { exact: true }).fill('90');
+    expect((await uiResponse(page, bPath, 'PUT', () => b.getByRole('button', { name: 'Save changes', exact: true }).click())).status()).toBe(200);
+    b = page.getByRole('article', { name: 'Updated B', exact: true });
+    await expect(b.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+    await b.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(b.getByText('No historical checks.', { exact: true })).toBeVisible();
+    await b.getByRole('button', { name: 'Hide history', exact: true }).click();
+    await reload();
+    await expect(b.getByText('90s interval · Enabled', { exact: true })).toBeVisible();
+    expect((await authoritative(bPath)).body.data.monitor).toMatchObject({ name: 'Updated B', interval: 90 });
+    for (const [button, state] of [['Pause', 'Paused'], ['Activate', 'Enabled']] as const) {
+      expect((await uiResponse(page, bPath, 'PUT', () => b.getByRole('button', { name: button, exact: true }).click())).status()).toBe(200);
+      await expect(b.getByText(`90s interval · ${state}`, { exact: true })).toBeVisible();
+    }
+    expect((await uiResponse(page, bPath, 'DELETE', () => b.getByRole('button', { name: 'Delete', exact: true }).click())).status()).toBe(200);
+    await expect(b).toHaveCount(0);
+    await reload();
+    await expect(b).toHaveCount(0);
+    expect((await authoritative(bPath)).status).toBe(404);
+    completed.push('B create/edit/pause/resume/delete and list/detail reload');
+
+    const a = page.getByRole('article', { name: 'Monitor A', exact: true });
+    const checked = await uiResponse(page, `${aPath}/checks`, 'POST', () => a.getByRole('button', { name: 'Run check', exact: true }).click());
+    expect(checked.status()).toBe(200);
+    const checkId = (await checked.json()).data.id as string;
+    await expect(a.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+    await a.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(a.locator(`tr[data-check-id="${checkId}"]`)).toHaveCount(1);
+    await a.getByRole('button', { name: 'Hide history', exact: true }).click();
+    await a.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(a.getByRole('table', { name: 'Check history' }).getByRole('row')).toHaveCount(2);
+    await reload();
+    await expect(a.getByText('SUCCEEDED', { exact: true })).toBeVisible();
+    await a.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(a.locator(`tr[data-check-id="${checkId}"]`)).toHaveCount(1);
+    const beforeA = await authoritative(aPath);
+    const beforeHistory = await authoritative(`${aPath}/checks`);
+    expect(beforeA.body.data.latestCheck.id).toBe(checkId);
+    expect(beforeHistory.body.data).toHaveLength(1);
+    completed.push('A one real Check, latest/history toggle and reload');
+    await a.getByRole('button', { name: 'Edit', exact: true }).click();
+    await fillCreate(page, 'Pending C');
+
+    intercepted = route => {
+      if (route.request().method() !== 'POST' || route.request().postDataJSON()?.name !== 'Pending C') return route.continue();
+      const ordinal = ++createPosts;
+      const work = (async () => {
+        const response = await safeRequest(() => route.fetch());
+        expect(response.status()).toBe(201);
+        if (ordinal === 1) { firstCommitted.resolve(); await release.promise; }
+        else secondCommitted.resolve();
+        await route.fulfill({ response });
+      })();
+      routes.push(work);
+      return work;
+    };
+    await page.route('**/api/monitors', intercepted);
+    const form = page.locator('section[aria-labelledby="create-title"] form');
+    const submit = () => form.evaluate(form => form.dispatchEvent(new Event('submit', { bubbles: true, cancelable: true })));
+    await submit();
+    await firstCommitted.promise;
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeDisabled();
+    await expect(page.getByRole('article', { name: 'Pending C', exact: true })).toHaveCount(0);
+    await submit();
+
+    if (baseline) {
+      await secondCommitted.promise;
+      const rows = (await authoritative('/api/monitors')).body.data;
+      const durableRows = rows.filter((row: { monitor: { name: string } }) => row.monitor.name === 'Pending C').length;
+      evidence.counterexample = { submitEvents: 2, createPosts, durableRows, requiredMaximum: 1,
+        firstRealResponseHeldAfterCommit: true };
+      expect(createPosts, 'Two submit events must create at most one request and durable row').toBe(1);
+      return;
+    }
+
+    await expect(page.getByRole('status').filter({ hasText: 'Creating monitor…' })).toBeVisible();
+    const injected: string[] = [];
+    await page.route(`**${aPath}`, route => {
+      const method = route.request().method();
+      if ((method === 'PUT' || method === 'DELETE') && !injected.includes(method)) {
+        expect(released).toBe(false);
+        injected.push(method);
+        return route.fulfill({ status: 500, contentType: 'application/json',
+          body: JSON.stringify({ error: { code: 'INTERNAL_ERROR', message: 'Fixed E06 failure' } }) });
+      }
+      return route.continue();
+    });
+    await a.getByLabel('Edit name', { exact: true }).fill('Rejected A');
+    await a.getByLabel('Edit interval (seconds)', { exact: true }).fill('90');
+    expect((await uiResponse(page, aPath, 'PUT', () => a.getByRole('button', { name: 'Save changes', exact: true }).click())).status()).toBe(500);
+    await expect(page.getByRole('alert')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(a.getByText('60s interval · Enabled', { exact: true })).toBeVisible();
+    expect(await authoritative(aPath)).toEqual(beforeA);
+    expect(await authoritative(`${aPath}/checks`)).toEqual(beforeHistory);
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeDisabled();
+    await expect(page.getByRole('status').filter({ hasText: 'Creating monitor…' })).toBeVisible();
+    expect((await uiResponse(page, aPath, 'DELETE', () => a.getByRole('button', { name: 'Delete', exact: true }).click())).status()).toBe(500);
+    await expect(page.getByRole('alert')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(a).toBeVisible();
+    expect(await authoritative(aPath)).toEqual(beforeA);
+    expect(await authoritative(`${aPath}/checks`)).toEqual(beforeHistory);
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeDisabled();
+    await expect(page.getByRole('status').filter({ hasText: 'Creating monitor…' })).toBeVisible();
+    expect(injected).toEqual(['PUT', 'DELETE']);
+    expect(createPosts).toBe(1);
+    completed.push('update500/delete500 retain authority and independent create pending');
+
+    released = true;
+    release.resolve();
+    await Promise.all(routes);
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeEnabled();
+    await expect(page.getByRole('status').filter({ hasText: 'Creating monitor…' })).toHaveCount(0);
+    await expect(page.getByRole('article', { name: 'Pending C', exact: true })).toHaveCount(1);
+    const rows = (await authoritative('/api/monitors')).body.data;
+    expect(rows.filter((row: { monitor: { name: string } }) => row.monitor.name === 'Pending C')).toHaveLength(1);
+    expect(createPosts).toBe(1);
+    await reload();
+    await expect(page.getByRole('article', { name: 'Pending C', exact: true })).toHaveCount(1);
+    expect(await authoritative(aPath)).toEqual(beforeA);
+    expect(await authoritative(`${aPath}/checks`)).toEqual(beforeHistory);
+    completed.push('release real201, one request/row, pending clears and reload remains canonical');
+    evidence.result = 'PASS';
+    evidence.delayedCreate = { submitEvents: 2, createPosts, durableRows: 1, realResponseStatus: 201 };
+    evidence.injectedFailures = { methods: injected, status: 500, whileCreateHeld: true, authoritativeRowsUnchanged: true };
+  } finally {
+    released = true;
+    release.resolve();
+    const settled = await Promise.allSettled(routes);
+    if (intercepted) await page.unroute('**/api/monitors', intercepted);
+    evidence.heldRoutesSettled = settled.every(result => result.status === 'fulfilled');
+    mkdirSync('output/e06', { recursive: true });
+    writeFileSync(`output/e06/${baseline ? 'baseline' : 'server-state'}.json`, `${JSON.stringify(evidence, null, 2)}\n`);
+  }
+});


