# E06 Server-State Cache와 Mutation Invalidation

## `test: freeze repeated-submit response barrier`

diff --git a/evidence/E06/baseline-reproducer.mjs b/evidence/E06/baseline-reproducer.mjs
new file mode 100644
index 0000000..a61c26f
--- /dev/null
+++ b/evidence/E06/baseline-reproducer.mjs
@@ -0,0 +1,73 @@
+// Run once at the frozen START; only this evidence directory may be new.
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync, spawn } from 'node:child_process';
+import { once } from 'node:events';
+import { readFile, writeFile } from 'node:fs/promises';
+import { createServer } from 'node:net';
+import { setTimeout as delay } from 'node:timers/promises';
+import { chromium } from '@playwright/test';
+import { buildApp } from '../../server/app.ts';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../server/database.ts';
+import { migrate } from '../../server/migrate.ts';
+import { prepareTestUsers } from '../../test/auth.ts';
+import { fixtureServer } from '../../test/fixture.ts';
+import { runServerStateScenario } from './browser-scenario.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), scenario.start);
+assert.equal(execFileSync('git', ['diff', '--name-only', 'HEAD', '--', 'server', 'test', 'app'], { encoding: 'utf8' }).trim(), '');
+const scenarioSha256 = createHash('sha256').update(await readFile(new URL('./scenario.json', import.meta.url))).digest('hex');
+const harnessSha256 = createHash('sha256').update(await readFile(new URL('./browser-scenario.ts', import.meta.url))).digest('hex');
+for (const port of [4311, 4312, 4313, 4314]) {
+  const guard = createServer();
+  await new Promise((resolve, reject) => { guard.once('error', reject); guard.listen(port, '127.0.0.1', resolve); });
+  await new Promise((resolve) => guard.close(resolve));
+}
+const config = { ...databaseConfig(), schema: scenario.baselineSchema };
+assert.match(config.schema, /^e06_[a-z_]+$/);
+const admin = databasePool(config);
+const fixture = fixtureServer();
+const app = buildApp(scenario.fixtureOrigin, config);
+let schemaOwned = false;
+let web;
+let browser;
+try {
+  await admin.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  schemaOwned = true;
+  await migrate(config);
+  const users = await prepareTestUsers(config);
+  await new Promise((resolve, reject) => { fixture.server.once('error', reject); fixture.server.listen(4311, '127.0.0.1', resolve); });
+  await app.listen({ host: '127.0.0.1', port: 4312 });
+  web = spawn(process.execPath, ['node_modules/next/dist/bin/next', 'dev', '--hostname', '127.0.0.1', '--port', '4313'], {
+    cwd: process.cwd(), env: { ...process.env, NEXT_TELEMETRY_DISABLED: '1' }, detached: true, stdio: 'ignore',
+  });
+  const deadline = Date.now() + scenario.limits.serverReadyTimeoutMs;
+  let ready = false;
+  while (Date.now() < deadline && web.exitCode === null) {
+    try { ready = (await fetch(`${scenario.browserOrigin}/monitors`)).ok; } catch {}
+    if (ready) break;
+    await delay(100);
+  }
+  assert.equal(ready, true, 'Owned Next process must become ready.');
+  browser = await chromium.launch();
+  const context = await browser.newContext({ baseURL: scenario.browserOrigin });
+  const page = await context.newPage();
+  page.setDefaultTimeout(scenario.limits.assertionTimeoutMs);
+  try {
+    const login = await page.request.post('/api/auth/login', { headers: { origin: scenario.browserOrigin }, data: users[0] });
+    assert.equal(login.status(), 200);
+  } catch { throw new Error('Baseline authentication failed; credentials suppressed.'); }
+  const evidence = { start: scenario.start, scenarioSha256, harnessSha256, codeUnmodified: true, ...await runServerStateScenario(page, 'baseline') };
+  await writeFile(new URL('./baseline.json', import.meta.url), JSON.stringify(evidence, null, 2) + '\n');
+  console.log(JSON.stringify(evidence));
+} finally {
+  await browser?.close();
+  if (web?.pid && web.exitCode === null) { const stopped = once(web, 'exit'); process.kill(-web.pid, 'SIGTERM'); await stopped; }
+  await app.close();
+  fixture.server.closeAllConnections();
+  await new Promise((resolve) => fixture.server.close(resolve));
+  if (schemaOwned) await admin.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`);
+  await admin.end();
+}
diff --git a/evidence/E06/baseline.json b/evidence/E06/baseline.json
new file mode 100644
index 0000000..7be20db
--- /dev/null
+++ b/evidence/E06/baseline.json
@@ -0,0 +1,18 @@
+{
+  "start": "33adeab16aab2b56ee2314e0324ab5c46cbf47c0",
+  "scenarioSha256": "be0ba3ef17b67140d17266028b257b41bb8ec2d4138beeb051f73681ecfd899b",
+  "harnessSha256": "c0ab1374a9f46658fcac642dc77095cb6abd39a8d204afe4095484c4303ddb56",
+  "codeUnmodified": true,
+  "result": "REPRODUCED",
+  "firstDecisiveFailure": "second submit starts another mutation while the first response is held",
+  "sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect": true,
+  "submitEvents": 2,
+  "csrfReadsAfterSecondSubmit": 2,
+  "forwardedCreateRequests": 2,
+  "authoritativeCreatedRows": 2,
+  "pendingVisibleBeforeSecondSubmit": true,
+  "heldResponses": 1,
+  "failedUpdates": 0,
+  "failedDeletes": 0,
+  "durationMs": 1426
+}
diff --git a/evidence/E06/browser-scenario.ts b/evidence/E06/browser-scenario.ts
new file mode 100644
index 0000000..8ad24be
--- /dev/null
+++ b/evidence/E06/browser-scenario.ts
@@ -0,0 +1,184 @@
+import { expect } from '@playwright/test';
+import type { Page } from '@playwright/test';
+import type { MonitorView } from '../../server/model.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+type Counters = { csrfReads: number; createRequests: number };
+
+async function createMonitor(page: Page, input: typeof scenario.monitors[number]): Promise<MonitorView> {
+  await page.getByLabel('Name', { exact: true }).fill(input.name);
+  await page.getByLabel('Endpoint URL', { exact: true }).fill(input.url);
+  await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(input.interval));
+  await page.getByLabel('Enabled', { exact: true }).check();
+  const response = page.waitForResponse((item) => item.url().endsWith('/api/monitors') && item.request().method() === 'POST');
+  await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+  const received = await response;
+  expect(received.status()).toBe(201);
+  const monitor: MonitorView = (await received.json()).data;
+  await expect(page.getByRole('article', { name: input.name, exact: true })).toBeVisible();
+  return monitor;
+}
+
+export async function runServerStateScenario(page: Page, mode: 'baseline' | 'verification') {
+  const started = performance.now();
+  await page.goto('/monitors');
+  const a = await createMonitor(page, scenario.monitors[0]);
+  const b = await createMonitor(page, scenario.monitors[1]);
+  let bArticle = page.getByRole('article', { name: b.name, exact: true });
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  await bArticle.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+  await bArticle.getByLabel('Edit name', { exact: true }).fill(scenario.update.name);
+  await bArticle.getByLabel('Edit interval (seconds)', { exact: true }).fill(String(scenario.update.interval));
+  await bArticle.getByRole('button', { name: 'Save changes', exact: true }).click();
+  bArticle = page.getByRole('article', { name: scenario.update.name, exact: true });
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await expect(bArticle.getByRole('region', { name: `Check history for ${scenario.update.name}` })).toBeVisible();
+  await bArticle.getByRole('button', { name: 'Hide history', exact: true }).click();
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  await page.reload();
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await bArticle.getByRole('button', { name: 'Pause monitor', exact: true }).click();
+  await expect(bArticle).toContainText('Paused · 90 seconds');
+  await bArticle.getByRole('button', { name: 'Enable monitor', exact: true }).click();
+  await expect(bArticle).toContainText('Enabled · 90 seconds');
+  await bArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(bArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  page.once('dialog', (dialog) => dialog.accept());
+  await bArticle.getByRole('button', { name: 'Delete monitor', exact: true }).click();
+  await expect(bArticle).toHaveCount(0);
+  await page.reload();
+  await expect(bArticle).toHaveCount(0);
+  expect((await page.request.get(`/api/monitors/${b.id}`)).status()).toBe(404);
+  expect((await page.request.get(`/api/monitors/${b.id}/checks`)).status()).toBe(404);
+  const aArticle = page.getByRole('article', { name: a.name, exact: true });
+  await aArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(aArticle.getByText('No historical checks.', { exact: true })).toBeVisible();
+  const checked = page.waitForResponse((item) => item.url().endsWith(`/api/monitors/${a.id}/checks`) && item.request().method() === 'POST');
+  await aArticle.getByRole('button', { name: 'Run check', exact: true }).click();
+  const check = (await (await checked).json()).data;
+  expect(check.state).toBe('SUCCEEDED');
+  expect(check.httpStatus).toBe(200);
+  await expect(aArticle.locator('dl')).toContainText('SUCCEEDED');
+  await expect(aArticle.locator('tbody tr')).toHaveCount(1);
+  await expect(aArticle.locator('tbody')).toContainText(check.id);
+  await aArticle.getByRole('button', { name: 'Hide history', exact: true }).click();
+  await aArticle.getByRole('button', { name: 'View history', exact: true }).click();
+  await expect(aArticle.locator('tbody')).toContainText(check.id);
+
+  // Count only operation names, never credentials, headers, or request bodies.
+  await page.evaluate(() => {
+    const state = window as typeof window & { e06Counters: Counters };
+    state.e06Counters = { csrfReads: 0, createRequests: 0 };
+    const originalFetch = window.fetch;
+    window.fetch = (...args) => {
+      const path = String(args[0]);
+      if (path === '/api/auth/csrf') state.e06Counters.csrfReads++;
+      if (path === '/api/monitors' && args[1]?.method === 'POST') state.e06Counters.createRequests++;
+      return originalFetch(...args);
+    };
+  });
+  const committed = Promise.withResolvers<void>();
+  const duplicateCommitted = Promise.withResolvers<void>();
+  const release = Promise.withResolvers<void>();
+  const firstDelivered = Promise.withResolvers<void>();
+  let forwardedCreates = 0;
+  let failedUpdates = 0;
+  let failedDeletes = 0;
+  await page.route('**/api/monitors', async (route) => {
+    if (route.request().method() !== 'POST') return route.continue();
+    const index = ++forwardedCreates;
+    const response = await route.fetch();
+    expect(response.status()).toBe(201);
+    if (index === 1) {
+      committed.resolve();
+      await release.promise;
+      await route.fulfill({ response });
+      firstDelivered.resolve();
+    } else {
+      await route.fulfill({ response });
+      duplicateCommitted.resolve();
+    }
+  });
+  try {
+    await page.getByLabel('Name', { exact: true }).fill(scenario.monitors[1].name);
+    await page.getByLabel('Endpoint URL', { exact: true }).fill(scenario.monitors[1].url);
+    await page.getByLabel('Interval (seconds)', { exact: true }).fill(String(scenario.monitors[1].interval));
+    await page.getByRole('button', { name: 'Create monitor', exact: true }).click();
+    await committed.promise;
+    await expect(page.getByRole('button', { name: 'Creating…', exact: true })).toBeDisabled();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(0);
+    const afterSubmit = await page.getByLabel('Name', { exact: true }).evaluate((input: HTMLInputElement) => {
+      input.form!.requestSubmit();
+      return (window as typeof window & { e06Counters: Counters }).e06Counters;
+    });
+    if (mode === 'baseline' && afterSubmit.csrfReads > scenario.barrier.requiredCsrfReads) {
+      await duplicateCommitted.promise;
+      const authoritative: MonitorView[] = (await (await page.request.get('/api/monitors')).json()).data;
+      return {
+        result: 'REPRODUCED', firstDecisiveFailure: 'second submit starts another mutation while the first response is held',
+        sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect: true,
+        submitEvents: scenario.barrier.submitEvents, csrfReadsAfterSecondSubmit: afterSubmit.csrfReads,
+        forwardedCreateRequests: forwardedCreates,
+        authoritativeCreatedRows: authoritative.filter((item) => item.name === scenario.monitors[1].name).length,
+        pendingVisibleBeforeSecondSubmit: true, heldResponses: 1,
+        failedUpdates, failedDeletes, durationMs: Math.round(performance.now() - started),
+      };
+    }
+    expect(afterSubmit.csrfReads).toBe(scenario.barrier.requiredCsrfReads);
+    expect(afterSubmit.createRequests).toBe(scenario.barrier.requiredCreateRequests);
+    await expect(page.getByRole('button', { name: 'Creating…', exact: true })).toBeDisabled();
+    const beforeFailures = (await (await page.request.get(`/api/monitors/${a.id}`)).json()).data;
+    const beforeHistory = (await (await page.request.get(`/api/monitors/${a.id}/checks`)).json()).data;
+    await page.route(`**/api/monitors/${a.id}`, async (route) => {
+      const method = route.request().method();
+      if (method === 'PUT') { failedUpdates++; return route.fulfill({ status: scenario.failure.status, json: scenario.failure.body }); }
+      if (method === 'DELETE') { failedDeletes++; return route.fulfill({ status: scenario.failure.status, json: scenario.failure.body }); }
+      return route.continue();
+    });
+    await aArticle.getByRole('button', { name: 'Edit monitor', exact: true }).click();
+    await aArticle.getByLabel('Edit name', { exact: true }).fill(scenario.rejectedUpdate.name);
+    await aArticle.getByLabel('Edit interval (seconds)', { exact: true }).fill(String(scenario.rejectedUpdate.interval));
+    await aArticle.getByRole('button', { name: 'Save changes', exact: true }).click();
+    const alert = page.getByRole('main').getByRole('alert');
+    await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(aArticle).toContainText('Enabled · 60 seconds');
+    await expect(aArticle.getByLabel('Edit name', { exact: true })).toHaveValue(scenario.rejectedUpdate.name);
+    await expect(aArticle.getByRole('button', { name: 'Save changes', exact: true })).toBeEnabled();
+    await aArticle.getByRole('button', { name: 'Cancel edit', exact: true }).click();
+    page.once('dialog', (dialog) => dialog.accept());
+    await aArticle.getByRole('button', { name: 'Delete monitor', exact: true }).click();
+    await expect(alert).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+    await expect(aArticle.getByRole('button', { name: 'Delete monitor', exact: true })).toBeEnabled();
+    await expect(aArticle.locator('tbody tr')).toHaveCount(1);
+    await expect(aArticle.locator('tbody')).toContainText(check.id);
+    expect((await (await page.request.get(`/api/monitors/${a.id}`)).json()).data).toEqual(beforeFailures);
+    expect((await (await page.request.get(`/api/monitors/${a.id}/checks`)).json()).data).toEqual(beforeHistory);
+    expect(failedUpdates).toBe(1);
+    expect(failedDeletes).toBe(1);
+    release.resolve();
+    await firstDelivered.promise;
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeEnabled();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(1);
+    expect(forwardedCreates).toBe(scenario.barrier.requiredCreateRequests);
+    await page.reload();
+    await expect(page.getByRole('article', { name: scenario.monitors[1].name, exact: true })).toHaveCount(1);
+    await expect(aArticle).toContainText('Enabled · 60 seconds');
+    const authoritative: MonitorView[] = (await (await page.request.get('/api/monitors')).json()).data;
+    expect(authoritative.filter((item) => item.name === scenario.monitors[1].name)).toHaveLength(scenario.barrier.requiredCreatedRows);
+    return {
+      result: mode === 'baseline' ? 'NOT_REPRODUCED' : 'PASS',
+      sequentialCreateUpdatePauseResumeDeleteAndCheckAlreadyCorrect: true,
+      submitEvents: scenario.barrier.submitEvents, csrfReadsAfterSecondSubmit: afterSubmit.csrfReads,
+      forwardedCreateRequests: forwardedCreates, authoritativeCreatedRows: scenario.barrier.requiredCreatedRows,
+      pendingVisibleBeforeSecondSubmit: true, heldResponses: 1, failedUpdates, failedDeletes,
+      failedMutationsPreservedAuthoritativeState: true, pendingClearedAfterRelease: true,
+      durationMs: Math.round(performance.now() - started),
+    };
+  } finally {
+    release.resolve();
+    await firstDelivered.promise;
+    await page.unrouteAll({ behavior: 'wait' });
+  }
+}
diff --git a/evidence/E06/scenario.json b/evidence/E06/scenario.json
new file mode 100644
index 0000000..b308348
--- /dev/null
+++ b/evidence/E06/scenario.json
@@ -0,0 +1,31 @@
+{
+  "thread": "E06",
+  "attempt": 1,
+  "start": "33adeab16aab2b56ee2314e0324ab5c46cbf47c0",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "baselineSchema": "e06_baseline",
+  "monitors": [
+    { "name": "Monitor A", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+    { "name": "Monitor B", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true }
+  ],
+  "update": { "name": "Updated B", "interval": 90 },
+  "rejectedUpdate": { "name": "Rejected A", "interval": 90 },
+  "failure": { "status": 500, "body": { "error": { "code": "INTERNAL_ERROR", "message": "The monitoring service could not complete the request." } } },
+  "sequence": [
+    "Create A and B through the authenticated browser. Inspect B in the list and its expanded history/detail panel.",
+    "Update B to Updated B / interval 90; hide/reopen its detail panel and reload the list.",
+    "Pause/resume B, delete B with its detail open, then reload and require B absent with API detail/history 404.",
+    "Open A history, complete exactly one manual check, and require that same result in the list and history after hide/reopen.",
+    "Create B again. Forward its first POST to the real API, hold only its response after commit at an explicit promise barrier, and require Creating pending with no B displayed.",
+    "While the response remains held, call the same form's requestSubmit exactly once. This is a second submit event even though the button is disabled. Require one CSRF read, one POST, and one authoritative B row.",
+    "While B creation is held, fail exactly one A PUT and one A DELETE at the browser response boundary with fixed 500 INTERNAL_ERROR; require A's old name, interval and history to remain and pending controls to recover.",
+    "Release the first create response explicitly. Require pending clear, exactly one visible B, and exactly one authoritative B row after reload."
+  ],
+  "detailMeaning": "The existing expanded Monitor article/history is the current detail view; no new URL routing or E07 URL state is introduced.",
+  "barrier": { "heldCreateResponses": 1, "point": "after real POST commit, before route.fulfill", "submitEvents": 2, "release": "explicit promise resolution, never a sleep", "requiredCsrfReads": 1, "requiredCreateRequests": 1, "requiredCreatedRows": 1 },
+  "limits": { "browserWorkers": 1, "browserRetries": 0, "assertionTimeoutMs": 5000, "testTimeoutMs": 30000, "serverReadyTimeoutMs": 90000, "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0 },
+  "baselinePolicy": "Run at unchanged START. Stop at the first decisive failure; if the duplicate submit emits a second CSRF request, wait only for that already submitted POST to observe its committed duplicate, then release the held response solely for cleanup."
+}


