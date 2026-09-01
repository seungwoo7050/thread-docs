## `fix: show aborted executions and refresh terminal history`

diff --git a/app/monitors/api.ts b/app/monitors/api.ts
index 4ad6487..c1918cb 100644
--- a/app/monitors/api.ts
+++ b/app/monitors/api.ts
@@ -1,6 +1,6 @@
 export type Monitor = { id: string; name: string; url: string; interval: number; enabled: boolean };
 export type CheckRun = {
-  id: string; monitorId: string; state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED'; httpStatus: number | null;
+  id: string; monitorId: string; state: 'QUEUED' | 'RUNNING' | 'SUCCEEDED' | 'FAILED' | 'ABORTED'; httpStatus: number | null;
   latencyMs: number | null; failureReason: 'HTTP_STATUS' | 'CONNECTION_FAILURE' | 'TIMEOUT' | null;
   startedAt: string | null; finishedAt: string | null;
 };
@@ -50,6 +50,10 @@ function isCheckRun(value: unknown): value is CheckRun {
     return value.finishedAt === null && value.httpStatus === null && value.latencyMs === null
       && value.failureReason === null && (value.state === 'QUEUED' ? value.startedAt === null : typeof value.startedAt === 'string');
   }
+  if (value.state === 'ABORTED') {
+    return typeof value.startedAt === 'string' && typeof value.finishedAt === 'string'
+      && value.httpStatus === null && value.latencyMs === null && value.failureReason === null;
+  }
   if (typeof value.startedAt !== 'string' || typeof value.finishedAt !== 'string' || typeof value.latencyMs !== 'number'
     || !Number.isInteger(value.latencyMs) || value.latencyMs < 0) return false;
   const status = value.httpStatus;
diff --git a/app/monitors/use-monitor-state.ts b/app/monitors/use-monitor-state.ts
index 2bdbcf2..25c1b61 100644
--- a/app/monitors/use-monitor-state.ts
+++ b/app/monitors/use-monitor-state.ts
@@ -72,7 +72,9 @@ export function useMonitorState(selection: HistorySelection | null, initial: Ini
           const monitors = current.monitors.map(row => {
             const observed = checks.find(check => check.monitorId === row.monitor.id && check.id === row.latestCheck?.id);
             if (!observed) return row;
-            if (observed.state === 'SUCCEEDED' || observed.state === 'FAILED') delete histories[row.monitor.id];
+            if (observed.state === 'SUCCEEDED' || observed.state === 'FAILED' || observed.state === 'ABORTED') {
+              delete histories[row.monitor.id];
+            }
             return { ...row, latestCheck: observed };
           });
           return { ...current, monitors, histories };
diff --git a/tests/browser/worker.spec.ts b/tests/browser/worker.spec.ts
index 424e110..8b89ac7 100644
--- a/tests/browser/worker.spec.ts
+++ b/tests/browser/worker.spec.ts
@@ -131,3 +131,98 @@ test('one persisted acceptance progresses in a separate worker while the browser
     writeFileSync(`${directory}/worker-browser.json`, JSON.stringify(evidence, null, 2) + '\n');
   }
 });
+
+test('an expired unknown execution becomes ABORTED in the current view and cached history', async ({ page }) => {
+  expect(process.env.E09_MANUAL_WORKER).toBe('1');
+  const output = 'output/phase-1/e11';
+  mkdirSync(output, { recursive: true });
+  const evidence: Record<string, unknown> = {
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/phase-1/E11/fixtures.md')).digest('hex'),
+    setup: 'explicit expired RUNNING fixture; actual normal worker recovery; no additional crash checkpoint',
+    result: 'INCOMPLETE',
+  };
+  let worker: ChildProcess | undefined;
+  let workerError = false;
+  let workerExitAwaited = false;
+  try {
+    const previous = (await (await safeRequest(() => page.request.get('/api/monitors'))).json()).data;
+    for (const row of previous) {
+      const deleteHeaders = await csrfHeaders(page.request);
+      expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, {
+        headers: deleteHeaders,
+      }))).status()).toBe(200);
+    }
+    let aId = '';
+    for (const [name, path] of [['A', '/ok'], ['B', '/fail']] as const) {
+      const headers = await csrfHeaders(page.request);
+      const response = await safeRequest(() => page.request.post('/api/monitors', {
+        headers, data: { name, url: `${fixture}${path}`, interval: 60, enabled: false },
+      }));
+      expect(response.status()).toBe(201);
+      if (name === 'A') aId = (await response.json()).data.monitor.id;
+    }
+    const headers = { ...(await csrfHeaders(page.request)), 'Idempotency-Key': 'e11-browser-recovery' };
+    const accepted = await safeRequest(() => page.request.post(`/api/monitors/${aId}/checks`, { headers }));
+    expect(accepted.status()).toBe(202);
+    const check = (await accepted.json()).data;
+    expect(check.state).toBe('QUEUED');
+    expect(check.id).toMatch(/^[0-9a-f-]{36}$/);
+    execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+      'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+      '--set', 'ON_ERROR_STOP=1', '--command', `UPDATE e04_browser.check_runs SET state='RUNNING',
+        queued_at='2026-08-01T00:00:00Z', started_at='2026-08-01T00:00:00Z',
+        claim_owner='00000000-0000-4000-8000-000000000111', lease_expires_at='2026-08-01T00:00:05Z'
+        WHERE id='${check.id}' AND state='QUEUED'`], { stdio: 'pipe' });
+    await page.goto('/monitors');
+    const a = page.getByRole('article', { name: 'A', exact: true });
+    const result = a.locator('[data-latest-check-id]');
+    await expect(result).toHaveAttribute('data-latest-check-id', check.id);
+    await expect(result.getByText('RUNNING', { exact: true })).toBeVisible();
+    await a.getByRole('button', { name: 'Show history', exact: true }).click();
+    await expect(a.getByText('No historical checks.', { exact: true })).toBeVisible();
+    evidence.beforeRecovery = { currentState: 'RUNNING', sameAcceptedId: true, cachedTerminalHistoryRows: 0 };
+
+    const log = openSync(`${output}/browser-worker.log`, 'w');
+    const { E04_ALICE_PASSWORD, E04_BOB_PASSWORD, ...runtime } = process.env;
+    worker = spawn('java', ['-jar', 'backend/target/monitor-api-0.0.1.jar', '--spring.profiles.active=worker',
+      '--spring.main.web-application-type=none', '--monitor.scheduler-enabled=false'], {
+      env: { ...runtime, DB_SCHEMA: 'e04_browser' }, stdio: ['ignore', log, log],
+    });
+    closeSync(log);
+    worker.once('error', () => { workerError = true; });
+    await expect.poll(() => {
+      expect(workerError || worker!.exitCode !== null || worker!.signalCode !== null).toBe(false);
+      return persisted()[0]?.state;
+    }, { intervals: [25], timeout: 30_000 }).toBe('ABORTED');
+    await expect(result.getByText('ABORTED', { exact: true })).toBeVisible();
+    await expect(result.getByText('HTTP —', { exact: true })).toBeVisible();
+    await expect(a.locator(`tr[data-check-id="${check.id}"]`)).toHaveCount(1);
+    const terminal = persisted();
+    expect(terminal.length).toBe(1);
+    expect(terminal[0].id).toBe(check.id);
+    expect(terminal[0].http_status).toBeNull();
+    expect(terminal[0].latency_ms).toBeNull();
+    expect(terminal[0].finished_at).not.toBeNull();
+    await expect(page.getByRole('main').getByRole('alert')).toHaveCount(0);
+    await page.reload();
+    await expect(result).toHaveAttribute('data-latest-check-id', check.id);
+    await expect(result.getByText('ABORTED', { exact: true })).toBeVisible();
+    await expect(a.locator(`tr[data-check-id="${check.id}"]`)).toHaveCount(1);
+    expect(persisted()).toEqual(terminal);
+    evidence.afterRecovery = { state: 'ABORTED', sameId: true, httpStatus: null, latencyMs: null,
+      terminalHistoryRows: 1, pollingInvalidatedCachedEmptyHistory: true, reloadRetainedTerminal: true,
+      actualWorkerPid: worker.pid };
+    evidence.result = 'PASS';
+  } finally {
+    if (worker && worker.exitCode === null && worker.signalCode === null) {
+      const exited = once(worker, 'exit');
+      worker.kill('SIGTERM');
+      const force = setTimeout(() => worker!.kill('SIGKILL'), 5000);
+      await exited;
+      clearTimeout(force);
+      workerExitAwaited = true;
+    }
+    evidence.cleanup = { workerExitAwaited };
+    writeFileSync(`${output}/browser.json`, JSON.stringify(evidence, null, 2) + '\n');
+  }
+});


