# E08 Server Rendering, Hydration과 접근 가능한 Browser 경계

## `test: freeze E08 authenticated production rendering counterexample`

diff --git a/evidence/E08/baseline.json b/evidence/E08/baseline.json
new file mode 100644
index 0000000..551cc72
--- /dev/null
+++ b/evidence/E08/baseline.json
@@ -0,0 +1,79 @@
+{
+  "mode": "baseline",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "productionBuild": true,
+  "hashes": {
+    "scenario.json": "a9126ebe618b4339878aa9ce94426d919375103a505a860e28ede747dbe390ca",
+    "fixture.mjs": "1527e16a8fc96286370619575ebb78e39111eab2b724b9b79c0a8ff434bf85da",
+    "browser-scenario.mjs": "49254a603cdb422072045219912861803a46d8c16abb6ffb10059f2c5a194043",
+    "run.mjs": "3d3d072104a1b47d31c86ea75399e594ae08ea24bab908b0cb88343b6100a62a"
+  },
+  "commands": [
+    {
+      "command": "npm run build (under fnm Node24.19.0)",
+      "exitCode": 0,
+      "durationMs": 3362
+    },
+    {
+      "command": "node test/fixture.ts",
+      "ready": true,
+      "readinessDurationMs": 80
+    },
+    {
+      "command": "node server/main.ts",
+      "ready": true,
+      "readinessDurationMs": 212
+    },
+    {
+      "command": "node node_modules/next/dist/bin/next start --hostname 127.0.0.1 --port 4313",
+      "ready": true,
+      "readinessDurationMs": 180
+    }
+  ],
+  "budget": {
+    "loadRuns": 0,
+    "automaticRetries": 0,
+    "parameterSweeps": 0
+  },
+  "result": "REPRODUCED",
+  "portsInitiallyFree": true,
+  "stage": "initial-production-html",
+  "initial": {
+    "status": 200,
+    "javaScriptEnabled": false,
+    "visibleMonitorCount": 0,
+    "historyIds": [],
+    "monitorNameInHtml": false,
+    "historyIdsInHtml": false,
+    "cacheControl": "s-maxage=31536000"
+  },
+  "decisiveFailure": "Authenticated production HTML does not render Monitor A and both existing CheckRuns before JavaScript.",
+  "cleanup": {
+    "children": [
+      {
+        "name": "production-next",
+        "code": 143,
+        "signal": null,
+        "forced": false,
+        "durationMs": 4
+      },
+      {
+        "name": "api",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 7
+      },
+      {
+        "name": "fixture",
+        "code": 0,
+        "signal": null,
+        "forced": false,
+        "durationMs": 3
+      }
+    ],
+    "schemaDropped": true,
+    "portsFree": true
+  },
+  "durationMs": 5189
+}
diff --git a/evidence/E08/browser-scenario.mjs b/evidence/E08/browser-scenario.mjs
new file mode 100644
index 0000000..dabd141
--- /dev/null
+++ b/evidence/E08/browser-scenario.mjs
@@ -0,0 +1,220 @@
+import assert from 'node:assert/strict';
+import { expect } from '@playwright/test';
+import scenario from './scenario.json' with { type: 'json' };
+
+export async function runRenderingScenario(browser, pool, users, mode, report, readServerLogs = null) {
+  const contexts = [];
+  const bodies = [];
+  const secrets = users.map((user) => user.password);
+  const errors = { hydration: 0, page: 0, console: 0 };
+  const expectedIds = [...scenario.checks].reverse().map((check) => check.id);
+  const privateMarkers = [scenario.monitor.name, ...scenario.checks.flatMap((check) => [check.id, check.startedAt, check.finishedAt])];
+  async function context(javaScriptEnabled = true) {
+    const current = await browser.newContext({ baseURL: scenario.browserOrigin, javaScriptEnabled });
+    contexts.push(current);
+    return current;
+  }
+  async function login(current, user) {
+    try {
+      const response = await current.request.post('/api/auth/login', { headers: { origin: scenario.browserOrigin }, data: user });
+      assert.equal(response.status(), 200);
+      const csrf = await current.request.get('/api/auth/csrf');
+      assert.equal(csrf.status(), 200);
+      secrets.push((await csrf.json()).data.csrfToken);
+      secrets.push(...(await current.cookies()).map((cookie) => cookie.value));
+    } catch { throw new Error('E08 authentication preparation failed; credential details suppressed.'); }
+  }
+  function watch(page) {
+    page.on('pageerror', () => { errors.page++; });
+    page.on('console', (message) => {
+      if (message.type() === 'error') errors.console++;
+      if (/hydration|hydrating|server rendered html|did not match/i.test(message.text())) errors.hydration++;
+    });
+  }
+  async function semantics(page) {
+    const structure = await page.evaluate(() => ({
+      main: document.querySelectorAll('main').length,
+      headings: Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6'), (heading) => ({ level: Number(heading.tagName.slice(1)), text: heading.textContent.trim() })),
+    }));
+    assert.equal(structure.main, 1);
+    assert.equal(structure.headings.filter((heading) => heading.level === 1).length, 1);
+    assert.equal(structure.headings[0].level, 1);
+    for (let i = 1; i < structure.headings.length; i++) assert.ok(structure.headings[i].level <= structure.headings[i - 1].level + 1, 'Heading levels must not skip.');
+    return structure;
+  }
+  async function tabTo(page, target) {
+    for (let count = 0; count <= scenario.thresholds.keyboardTabMaximumPerTarget; count++) {
+      if (await target.evaluate((element) => element === document.activeElement)) return count;
+      if (count < scenario.thresholds.keyboardTabMaximumPerTarget) await page.keyboard.press('Tab');
+    }
+    throw new Error('Fixed keyboard Tab budget exhausted.');
+  }
+  function assertNoPrivateData(text) {
+    assert.ok(privateMarkers.every((marker) => !text.includes(marker)), 'Another user response must not expose Alice record data.');
+  }
+  try {
+    assert.equal(browser.version(), scenario.browser.chromium);
+    report.stage = 'initial-production-html';
+    const noJs = await context(false);
+    await login(noJs, users[0]);
+    const before = await noJs.newPage();
+    const initialResponse = await before.goto(scenario.primaryRoute, { waitUntil: 'load' });
+    assert.equal(initialResponse.status(), 200);
+    const initialHtml = await initialResponse.text();
+    bodies.push(initialHtml);
+    const initialIds = await before.locator('tbody tr td:first-child').allTextContents();
+    report.initial = {
+      status: initialResponse.status(), javaScriptEnabled: false,
+      visibleMonitorCount: await before.getByRole('article', { name: scenario.monitor.name, exact: true }).count(),
+      historyIds: initialIds,
+      monitorNameInHtml: initialHtml.includes(scenario.monitor.name),
+      historyIdsInHtml: expectedIds.every((id) => initialHtml.includes(id)),
+      cacheControl: initialResponse.headers()['cache-control'] ?? null,
+    };
+    if (report.initial.visibleMonitorCount !== 1 || JSON.stringify(initialIds) !== JSON.stringify(expectedIds)) {
+      report.result = mode === 'baseline' ? 'REPRODUCED' : 'FAILED';
+      report.decisiveFailure = 'Authenticated production HTML does not render Monitor A and both existing CheckRuns before JavaScript.';
+      if (mode === 'baseline') return report;
+      assert.fail(report.decisiveFailure);
+    }
+    assert.match(report.initial.cacheControl ?? '', /no-store/);
+    report.initial.structure = await semantics(before);
+    const initialText = await before.getByRole('main').innerText();
+
+    report.stage = 'hydration';
+    const alice = await context();
+    await alice.addCookies(await noJs.cookies()); // In memory only; never storageState.
+    const page = await alice.newPage();
+    watch(page);
+    let initialDataReads = 0;
+    const countRead = (request) => {
+      if (request.method() === 'GET' && /^\/api\/monitors(?:\/|$)/.test(new URL(request.url()).pathname)) initialDataReads++;
+    };
+    page.on('request', countRead);
+    const hydratedResponse = await page.goto(scenario.primaryRoute, { waitUntil: 'load' });
+    bodies.push(await hydratedResponse.text());
+    await expect(page.getByRole('article', { name: scenario.monitor.name, exact: true })).toBeVisible();
+    await expect(page.locator('tbody tr td:first-child')).toHaveText(expectedIds);
+    assert.equal(await page.getByRole('main').innerText(), initialText);
+    assert.deepEqual(await semantics(page), report.initial.structure);
+    // This keyboard action proves React attached handlers, without a test-only
+    // hydration flag or a delay. It is the first interaction after comparison.
+    await tabTo(page, page.getByRole('button', { name: 'Hide history', exact: true }));
+    await page.keyboard.press('Enter');
+    await expect(page.getByRole('button', { name: 'View history', exact: true })).toBeVisible();
+    page.off('request', countRead);
+    assert.equal(initialDataReads, scenario.thresholds.initialBrowserDataReads);
+    assert.deepEqual(errors, { hydration: 0, page: 0, console: 0 });
+    report.hydration = { identicalInitialMainText: true, identicalHeadingStructure: true, initialDataReads, keyboardHandlerAttached: true, errors: { ...errors } };
+
+    report.stage = 'privacy';
+    const bob = await context(false);
+    await login(bob, users[1]);
+    const anonymous = await context(false);
+    report.privacy = [];
+    for (const [identity, current] of [['alice', noJs], ['bob', bob], ['anonymous', anonymous]]) {
+      const html = await current.request.get(scenario.primaryRoute, { maxRedirects: 0 });
+      const htmlBody = await html.text();
+      const rsc = await current.request.get(scenario.primaryRoute, { headers: { rsc: '1' }, maxRedirects: 0 });
+      const rscBody = await rsc.text();
+      bodies.push(htmlBody, rscBody);
+      if (identity === 'alice') {
+        assert.equal(html.status(), 200);
+        assert.ok(privateMarkers.every((marker) => htmlBody.includes(marker) && rscBody.includes(marker)));
+      } else {
+        assertNoPrivateData(htmlBody);
+        assertNoPrivateData(rscBody);
+        if (identity === 'bob') {
+          assert.equal(html.status(), 200);
+          assert.ok(htmlBody.includes('No monitors yet.'));
+        } else {
+          assert.ok([303, 307, 308].includes(html.status()));
+          assert.equal(html.headers().location, '/login');
+        }
+      }
+      if (identity !== 'anonymous') {
+        assert.match(html.headers()['cache-control'] ?? '', /no-store/);
+        assert.match(rsc.headers()['cache-control'] ?? '', /no-store/);
+        assert.match(rsc.headers()['content-type'] ?? '', /text\/x-component/);
+      }
+      report.privacy.push({ identity, htmlStatus: html.status(), rscStatus: rsc.status(), noForeignRecordData: identity !== 'alice', noStore: /no-store/.test(html.headers()['cache-control'] ?? ''), rscNoStore: /no-store/.test(rsc.headers()['cache-control'] ?? '') });
+    }
+
+    report.stage = 'keyboard';
+    await page.goto('/monitors');
+    const tabCounts = [];
+    tabCounts.push(await tabTo(page, page.getByLabel('Name', { exact: true })));
+    await page.keyboard.type(scenario.create.name);
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('Endpoint URL', { exact: true })).toBeFocused();
+    await page.keyboard.press('ControlOrMeta+A');
+    await page.keyboard.type(scenario.create.url);
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('Interval (seconds)', { exact: true })).toBeFocused();
+    await page.keyboard.press('ControlOrMeta+A');
+    await page.keyboard.type(String(scenario.create.interval));
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('Enabled', { exact: true })).toBeFocused();
+    await expect(page.getByLabel('Enabled', { exact: true })).toBeChecked();
+    await page.keyboard.press('Tab');
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeFocused();
+    const created = page.waitForResponse((response) => new URL(response.url()).pathname === '/api/monitors' && response.request().method() === 'POST');
+    await page.keyboard.press('Enter');
+    assert.equal((await created).status(), 201);
+    await expect(page.getByRole('article', { name: scenario.create.name, exact: true })).toBeVisible();
+    const article = page.getByRole('article', { name: scenario.monitor.name, exact: true });
+    tabCounts.push(await tabTo(page, article.getByRole('button', { name: 'View history', exact: true })));
+    await page.keyboard.press('Enter');
+    const history = article.getByRole('region', { name: `Check history for ${scenario.monitor.name}` });
+    await expect(history.locator('tbody tr td:first-child')).toHaveText(expectedIds);
+    tabCounts.push(await tabTo(page, history.getByLabel('History state', { exact: true })));
+    await page.keyboard.press('ArrowDown');
+    await page.keyboard.press('Enter');
+    await expect(history.getByLabel('History state', { exact: true })).toHaveValue('SUCCEEDED');
+    await expect(history.locator('tbody tr td:first-child')).toHaveText([scenario.checks[0].id]);
+    await page.keyboard.press('Home');
+    await page.keyboard.press('Enter');
+    await expect(history.getByLabel('History state', { exact: true })).toHaveValue('');
+    await expect(history.locator('tbody tr td:first-child')).toHaveText(expectedIds);
+    report.keyboard = { pointerActions: 0, createdMonitors: 1, tabCounts, openedDetailHistory: true, selectedSuccessAndAll: true };
+
+    report.stage = 'accessibility';
+    const { default: axe } = await import('axe-core');
+    assert.equal(axe.version, scenario.accessibility.version);
+    const loginContext = await context();
+    const loginPage = await loginContext.newPage();
+    watch(loginPage);
+    report.accessibility = [];
+    for (const [screen, current, route] of [['login', loginPage, '/login'], ['list', page, '/monitors'], ['detail', page, scenario.primaryRoute]]) {
+      await current.goto(route, { waitUntil: 'load' });
+      const structure = await semantics(current);
+      await current.addScriptTag({ content: axe.source });
+      const result = await current.evaluate(async () => {
+        const result = await window.axe.run(document);
+        const safe = (items) => items.map((item) => ({ id: item.id, impact: item.impact, nodes: item.nodes.map((node) => node.target) }));
+        return { violations: safe(result.violations), incomplete: safe(result.incomplete), passedRules: result.passes.length };
+      });
+      report.accessibility.push({ screen, tool: `axe-core@${axe.version}`, structure, ...result });
+      assert.equal(result.violations.filter((violation) => violation.impact === 'serious').length, scenario.accessibility.seriousMaximum);
+      assert.equal(result.violations.filter((violation) => violation.impact === 'critical').length, scenario.accessibility.criticalMaximum);
+    }
+    report.stage = 'final-invariants';
+    const stored = (await pool.query('SELECT id, name FROM monitors WHERE owner_user_id = $1 ORDER BY name', [scenario.users[0].id])).rows;
+    assert.deepEqual(stored.map((monitor) => monitor.name), [scenario.monitor.name, scenario.create.name]);
+    assert.equal((await pool.query('SELECT count(*)::int AS count FROM monitors WHERE owner_user_id = $1', [scenario.users[1].id])).rows[0].count, 0);
+    assert.deepEqual((await pool.query('SELECT id FROM check_runs WHERE monitor_id = $1 ORDER BY finished_at DESC, id DESC', [scenario.monitor.id])).rows.map((check) => check.id), expectedIds);
+    secrets.push(...(await pool.query('SELECT password_hash FROM users WHERE id = ANY($1::uuid[])', [scenario.users.map((user) => user.id)])).rows.map((row) => row.password_hash));
+    secrets.push(...(await pool.query('SELECT token_hash FROM sessions WHERE user_id = ANY($1::uuid[])', [scenario.users.map((user) => user.id)])).rows.map((row) => row.token_hash));
+    assert.ok(bodies.every((body) => secrets.every((secret) => !body.includes(secret))), 'HTML/RSC must not serialize credential values.');
+    assert.ok(await page.evaluate(() => localStorage.length === 0 && sessionStorage.length === 0));
+    assert.ok(await page.evaluate(() => !document.cookie.includes('wse_session=')));
+    if (readServerLogs) assert.ok(secrets.every((secret) => !readServerLogs().includes(secret)), 'Server logs must not contain credentials.');
+    assert.deepEqual(errors, { hydration: 0, page: 0, console: 0 });
+    report.final = { aliceMonitors: 2, bobMonitors: 0, unchangedCheckIds: expectedIds, secretValuesInHtmlOrRsc: 0, browserStorageEntries: 0, httpOnlyCookieHidden: true, serverLogSecrets: readServerLogs ? 0 : 'checked by standalone production runner', errors };
+    report.result = mode === 'baseline' ? 'NOT_REPRODUCED' : 'PASS';
+    report.stage = 'complete';
+    return report;
+  } finally {
+    for (const current of contexts.reverse()) await current.close();
+  }
+}
diff --git a/evidence/E08/fixture.mjs b/evidence/E08/fixture.mjs
new file mode 100644
index 0000000..194742a
--- /dev/null
+++ b/evidence/E08/fixture.mjs
@@ -0,0 +1,29 @@
+import { randomBytes } from 'node:crypto';
+import { hashPassword } from '../../server/password.ts';
+import { checkRunToValues, monitorToValues } from '../../server/mapping.ts';
+import scenario from './scenario.json' with { type: 'json' };
+
+// The public data is fully fixed. Credentials are fresh, process-local values.
+export async function insertRenderingFixture(pool) {
+  const users = [];
+  for (const user of scenario.users) {
+    const password = randomBytes(32).toString('base64url');
+    const encoded = await hashPassword(password);
+    await pool.query('INSERT INTO users (id, username, password_hash) VALUES ($1, $2, $3)', [user.id, user.username, encoded]);
+    users.push({ username: user.username, password });
+  }
+  await pool.query(`INSERT INTO monitors
+    (id, name, url, interval_seconds, enabled, created_at, updated_at, owner_user_id)
+    VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`, [...monitorToValues(scenario.monitor), scenario.users[0].id]);
+  for (const check of scenario.checks) {
+    await pool.query(`INSERT INTO check_runs
+      (id, monitor_id, trigger, state, http_status, latency_ms, failure_reason, started_at, finished_at)
+      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)`, checkRunToValues(check));
+  }
+  return users;
+}
+
+export async function removeRenderingFixture(pool) {
+  await pool.query('DELETE FROM monitors WHERE owner_user_id = ANY($1::uuid[])', [scenario.users.map((user) => user.id)]);
+  await pool.query('DELETE FROM users WHERE id = ANY($1::uuid[])', [scenario.users.map((user) => user.id)]);
+}
diff --git a/evidence/E08/run.mjs b/evidence/E08/run.mjs
new file mode 100644
index 0000000..2dc20c5
--- /dev/null
+++ b/evidence/E08/run.mjs
@@ -0,0 +1,114 @@
+import assert from 'node:assert/strict';
+import { createHash } from 'node:crypto';
+import { execFileSync, spawn } from 'node:child_process';
+import { createServer } from 'node:net';
+import { mkdir, readFile, writeFile } from 'node:fs/promises';
+import { chromium } from '@playwright/test';
+import { databaseConfig, databasePool, schemaIdentifier } from '../../server/database.ts';
+import { migrate } from '../../server/migrate.ts';
+import { insertRenderingFixture } from './fixture.mjs';
+import { runRenderingScenario } from './browser-scenario.mjs';
+import scenario from './scenario.json' with { type: 'json' };
+
+const mode = process.argv[2];
+assert.ok(mode === 'baseline' || mode === 'verification');
+assert.equal(process.versions.node, '24.19.0');
+assert.equal(execFileSync('git', ['branch', '--show-current'], { encoding: 'utf8' }).trim(), 'track/fundamentals-fastify');
+assert.equal((await readFile('SPEC_REVISION', 'utf8')).trim(), scenario.specRevision);
+if (mode === 'baseline') {
+  assert.equal(execFileSync('git', ['rev-parse', 'HEAD'], { encoding: 'utf8' }).trim(), scenario.start);
+  assert.equal(execFileSync('git', ['diff', '--name-only', 'HEAD'], { encoding: 'utf8' }).trim(), '');
+}
+const started = performance.now();
+const report = { mode, start: scenario.start, productionBuild: true, hashes: {}, commands: [], budget: scenario.budget, result: 'FAILED_BEFORE_SCENARIO' };
+for (const name of ['scenario.json', 'fixture.mjs', 'browser-scenario.mjs', 'run.mjs']) {
+  report.hashes[name] = createHash('sha256').update(await readFile(new URL(name, import.meta.url))).digest('hex');
+}
+const children = [];
+let browser;
+let pool;
+let ownedSchema = false;
+const config = { ...databaseConfig(), schema: scenario.schema };
+assert.match(config.schema, /^e08_[a-z_]+$/);
+
+async function requireFreePorts() {
+  for (const port of scenario.ports) {
+    const guard = createServer();
+    await new Promise((resolve, reject) => { guard.once('error', reject); guard.listen(port, '127.0.0.1', resolve); });
+    await new Promise((resolve, reject) => guard.close((error) => error ? reject(error) : resolve()));
+  }
+}
+async function build() {
+  const began = performance.now();
+  const child = spawn('npm', ['run', 'build'], { env: { ...process.env, NEXT_TELEMETRY_DISABLED: '1' }, stdio: ['ignore', 'pipe', 'pipe'] });
+  let output = '';
+  child.stdout.on('data', (chunk) => { output += chunk; });
+  child.stderr.on('data', (chunk) => { output += chunk; });
+  const code = await new Promise((resolve, reject) => { child.once('error', reject); child.once('close', resolve); });
+  report.commands.push({ command: 'npm run build (under fnm Node24.19.0)', exitCode: code, durationMs: Math.round(performance.now() - began) });
+  if (code !== 0) {
+    // No accounts have been prepared yet, so this build-only diagnostic cannot
+    // include the scenario's passwords, cookies or session values.
+    console.error(output);
+    throw new Error('Production build failed before fixture preparation.');
+  }
+}
+async function start(name, args, readiness) {
+  const began = performance.now();
+  const child = spawn(process.execPath, args, { env: { ...process.env, DATABASE_SCHEMA: config.schema, NEXT_TELEMETRY_DISABLED: '1' }, stdio: ['ignore', 'pipe', 'pipe'] });
+  const record = { name, child, output: '', exited: null };
+  record.exited = new Promise((resolve) => { child.once('close', (code, signal) => resolve({ code, signal })); });
+  children.push(record);
+  await new Promise((resolve, reject) => {
+    const timer = setTimeout(() => reject(new Error(`${name} did not become ready within the fixed deadline.`)), scenario.thresholds.readinessTimeoutMs);
+    const append = (chunk) => {
+      record.output += chunk;
+      if (record.output.includes(readiness)) { clearTimeout(timer); resolve(); }
+    };
+    child.stdout.on('data', append);
+    child.stderr.on('data', append);
+    child.once('error', () => { clearTimeout(timer); reject(new Error(`${name} could not start.`)); });
+    child.once('close', () => { clearTimeout(timer); reject(new Error(`${name} exited before readiness.`)); });
+  });
+  report.commands.push({ command: `node ${args.join(' ')}`, ready: true, readinessDurationMs: Math.round(performance.now() - began) });
+}
+
+try {
+  await requireFreePorts();
+  report.portsInitiallyFree = true;
+  await build();
+  pool = databasePool(config);
+  await pool.query(`CREATE SCHEMA ${schemaIdentifier(config.schema)}`);
+  ownedSchema = true;
+  await migrate(config);
+  const users = await insertRenderingFixture(pool);
+  await start('fixture', ['test/fixture.ts'], 'Controlled fixture listening on loopback.');
+  await start('api', ['server/main.ts'], 'Monitor API listening on loopback.');
+  await start('production-next', ['node_modules/next/dist/bin/next', 'start', '--hostname', '127.0.0.1', '--port', '4313'], 'Ready in');
+  browser = await chromium.launch();
+  await runRenderingScenario(browser, pool, users, mode, report, () => children.map((child) => child.output).join('\n'));
+  assert.ok(mode === 'baseline' ? ['REPRODUCED', 'NOT_REPRODUCED'].includes(report.result) : report.result === 'PASS');
+} catch (error) {
+  report.failure = { stage: report.stage ?? 'preparation', message: error.message };
+  process.exitCode = 1;
+} finally {
+  if (browser) await browser.close();
+  report.cleanup = { children: [], schemaDropped: false, portsFree: false };
+  for (const record of children.reverse()) {
+    const began = performance.now();
+    let forced = false;
+    const timer = setTimeout(() => { forced = true; record.child.kill('SIGKILL'); }, scenario.thresholds.shutdownTimeoutMs);
+    record.child.kill('SIGTERM');
+    const exit = await record.exited;
+    clearTimeout(timer);
+    report.cleanup.children.push({ name: record.name, ...exit, forced, durationMs: Math.round(performance.now() - began) });
+  }
+  if (ownedSchema) { await pool.query(`DROP SCHEMA ${schemaIdentifier(config.schema)} CASCADE`); report.cleanup.schemaDropped = true; }
+  if (pool) await pool.end();
+  await requireFreePorts();
+  report.cleanup.portsFree = true;
+  report.durationMs = Math.round(performance.now() - started);
+  await mkdir('output/e08', { recursive: true });
+  await writeFile(`output/e08/${mode}.json`, JSON.stringify(report, null, 2) + '\n');
+  console.log(JSON.stringify(report, null, 2));
+}
diff --git a/evidence/E08/scenario.json b/evidence/E08/scenario.json
new file mode 100644
index 0000000..8a48b10
--- /dev/null
+++ b/evidence/E08/scenario.json
@@ -0,0 +1,64 @@
+{
+  "thread": "E08",
+  "start": "319f9aa027a3e88cd90afe8a9096276cac2ab7a6",
+  "specRevision": "0a006589477f8ae47bad3faa5510c999cff85ee4",
+  "schema": "e08_browser",
+  "apiOrigin": "http://127.0.0.1:4312",
+  "browserOrigin": "http://127.0.0.1:4313",
+  "fixtureOrigin": "http://127.0.0.1:4311",
+  "ports": [4311, 4312, 4313, 4314],
+  "users": [
+    { "id": "08000000-0000-4000-8000-000000000001", "username": "alice-e08" },
+    { "id": "08000000-0000-4000-8000-000000000002", "username": "bob-e08" }
+  ],
+  "credentials": "Fresh independent random passwords and session values exist only in process memory; never serialized or logged.",
+  "monitor": {
+    "id": "08000000-0000-4000-9000-000000000001",
+    "name": "E08 Monitor A",
+    "url": "http://127.0.0.1:4311/ok",
+    "interval": 60,
+    "enabled": true,
+    "createdAt": "2026-07-08T00:00:00.000Z",
+    "updatedAt": "2026-07-08T00:00:00.000Z"
+  },
+  "checks": [
+    {
+      "id": "08000000-0000-4000-a000-000000000001",
+      "monitorId": "08000000-0000-4000-9000-000000000001",
+      "trigger": "MANUAL",
+      "state": "SUCCEEDED",
+      "httpStatus": 200,
+      "latencyMs": 1,
+      "failureReason": null,
+      "startedAt": "2026-07-08T00:00:01.000Z",
+      "finishedAt": "2026-07-08T00:00:01.001Z"
+    },
+    {
+      "id": "08000000-0000-4000-a000-000000000002",
+      "monitorId": "08000000-0000-4000-9000-000000000001",
+      "trigger": "MANUAL",
+      "state": "FAILED",
+      "httpStatus": 503,
+      "latencyMs": 2,
+      "failureReason": "HTTP_STATUS",
+      "startedAt": "2026-07-08T00:00:02.000Z",
+      "finishedAt": "2026-07-08T00:00:02.002Z"
+    }
+  ],
+  "primaryRoute": "/monitors?monitor=08000000-0000-4000-9000-000000000001&limit=20",
+  "create": { "name": "E08 Monitor B", "url": "http://127.0.0.1:4311/ok", "interval": 60, "enabled": true },
+  "sequence": [
+    "Start only production Next build with real Fastify, fixture and PostgreSQL.",
+    "Authenticate Alice in a JavaScript-disabled Chromium context, request primaryRoute once and inspect visible initial Monitor A and both terminal rows. Baseline stops at the first decisive missing SSR assertion.",
+    "Hydrate primaryRoute in a fresh Chromium context sharing Alice cookies only in memory. Compare the initial main text, IDs and hierarchy; require zero initial browser Monitor/history GETs and zero hydration/page errors.",
+    "Read the same HTML and RSC URL as Alice, Bob (zero Monitors) and anonymous. Check no Alice record fields in Bob/anonymous bodies, no credentials in any body, and no-store for authenticated HTML/RSC. Request-echoed Monitor ID is not private record data.",
+    "Go to the list and use only Tab, text keys, Enter and arrow keys to create B once, open A history and select SUCCEEDED then All states.",
+    "Scan login, list and selected A detail/history with axe-core and assert one main, one h1, and no skipped heading level.",
+    "Confirm PostgreSQL still has A/B and exactly the original two checks; scan process logs for the in-memory credential values; close all contexts and owned processes and drop only the owned schema."
+  ],
+  "browser": { "playwright": "1.62.1", "chromium": "151.0.7922.34", "revision": 1234, "workers": 1 },
+  "accessibility": { "package": "axe-core", "version": "4.10.3", "rules": "all default rules; no exclusions", "seriousMaximum": 0, "criticalMaximum": 0, "recordLowerSeveritiesAndIncomplete": true },
+  "thresholds": { "baselineRuns": 1, "initialBrowserDataReads": 0, "hydrationErrors": 0, "pageErrors": 0, "keyboardTabMaximumPerTarget": 40, "testTimeoutMs": 30000, "readinessTimeoutMs": 30000, "shutdownTimeoutMs": 10000 },
+  "budget": { "loadRuns": 0, "automaticRetries": 0, "parameterSweeps": 0 },
+  "artifacts": { "rawHtml": false, "rawRsc": false, "logs": false, "credentials": false, "traces": false, "screenshots": false, "videos": false, "storageState": false }
+}


