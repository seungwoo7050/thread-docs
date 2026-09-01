# E08 Server Rendering, Hydration과 접근 가능한 Browser 경계

## `test(rendering): freeze phase-1 E08 SSR baseline`

diff --git a/SPEC_REVISION b/SPEC_REVISION
index a7af57d..42d5d11 100644
--- a/SPEC_REVISION
+++ b/SPEC_REVISION
@@ -1 +1 @@
-0a006589477f8ae47bad3faa5510c999cff85ee4
+2ada57a71cd34fa2fae9809415c362a8bbfcdf02
diff --git a/TRACK.md b/TRACK.md
index c02d207..9fa02c8 100644
--- a/TRACK.md
+++ b/TRACK.md
@@ -1,6 +1,8 @@
 # Industry / Spring
 
-Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`
+Profile: `phase-1` (E01–E12 → E20 → profile E24; phase-2 deferred, E25 not selected).
+
+Spec revision: `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`
 
 E06 is a local development product: Next.js/React renders login, logout, Monitor creation, editing, pause/activation, deletion and Check history. Spring Security authenticates browser sessions; PostgreSQL stores salted account passwords, Monitors and every completed CheckRun. A manual check is synchronous. Monitor and CheckRun access is scoped to the signed-in user. One React hook owns the page's server data and request state. There is no signup, scheduler, worker, Redis, broker, or production application container.
 
diff --git a/evidence/phase-1/E08/baseline.json b/evidence/phase-1/E08/baseline.json
new file mode 100644
index 0000000..58bd0c2
--- /dev/null
+++ b/evidence/phase-1/E08/baseline.json
@@ -0,0 +1,15 @@
+{
+  "start": "50f633fe194c079db0cc438cac1a517702aef234",
+  "fixtureSha256": "77beb223b83fc659830bf090eb5c3a469f7f14bce5dc3e12275587d3a89b2542",
+  "production": true,
+  "baseline": true,
+  "completed": [],
+  "initialHtml": {
+    "status": 200,
+    "monitorRendered": false,
+    "renderedHistoryIds": [],
+    "mainCount": 1,
+    "h1Count": 0,
+    "beforeBrowserJavaScript": true
+  }
+}
diff --git a/evidence/phase-1/E08/fixtures.md b/evidence/phase-1/E08/fixtures.md
new file mode 100644
index 0000000..8e51566
--- /dev/null
+++ b/evidence/phase-1/E08/fixtures.md
@@ -0,0 +1,85 @@
+# Phase-1 E08 frozen SSR, privacy, hydration and accessibility case
+
+Frozen before baseline execution or product edits. Attempt1.
+START `50f633fe194c079db0cc438cac1a517702aef234`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Branch `track/industry-spring`; profile `phase-1`.
+Metadata adopts this revision in the first E08 commit; baseline product remains START.
+
+## Fixed data and environment
+
+Use the existing owned `e04_browser` PostgreSQL fixture, runtime-only Alice/Bob
+credentials, session/CSRF helpers, fixture4321, API4322 and UI4323. Clear only the
+two fixture users' owned rows. Alice has one enabled `Monitor A`,
+`http://127.0.0.1:4321/ok`, interval60. Bob has zero Monitors.
+Create the Monitor through the real authenticated API; its UUID is a runtime
+identifier used consistently in all requests. Insert two existing terminal runs:
+
+| ID | Trigger/state/status/reason | Latency | Started and finished |
+| --- | --- | --- | --- |
+| `00000000-0000-4000-8000-000000000081` | MANUAL/SUCCEEDED/200/null | 1ms | `2026-08-28T00:00:00.000Z` |
+| `00000000-0000-4000-8000-000000000082` | MANUAL/SUCCEEDED/200/null | 1ms | `2026-08-28T00:00:01.000Z` |
+
+The primary URL is `/monitors?history=<Monitor A UUID>&limit=3` for every actor.
+Use pinned production Next build/start and Chromium revision1234, not dev mode.
+Use exactly one automated accessibility tool, dev-only `axe-core@4.11.0`, with
+its default rule set and no exclusion/disable list. No new production dependency.
+
+## Single unchanged-product baseline
+
+Start the production browser harness once. After the fixed fixture setup, fetch
+Alice's primary HTML through her request context before browser JavaScript runs.
+Record only status and presence/count flags for the Monitor article, two history
+rows and main/heading structure. Require Monitor A and rows82,81 in server HTML.
+If absent, record the decisive absence and stop this baseline; do not run extra
+exploratory SSR, hydration or accessibility sequences. All passing existing
+behavior remains unchanged; no other defect is presumed.
+
+## Post-change sequence and fixed barriers
+
+1. Fetch the same Alice HTML before JavaScript. Require the Monitor, latest82,
+   rows82,81 and semantic main/heading structure. Authenticated HTML must not use
+   shared caching. Server fetches must be request/session-scoped and no-store.
+2. Fetch that exact URL as Bob and anonymously. Neither may include Alice's
+   Monitor article/name or either CheckRun payload. An anonymous response must
+   redirect to `/login` (or preserve the existing401 boundary). A supplied URL ID
+   is an input, not evidence of access; no returned private record is permitted.
+3. Check each raw HTML/RSC response in memory for the runtime password, session
+   cookie and known CSRF values; all must be absent. Retain only booleans/counts,
+   never raw private HTML, response bodies, cookie values or token material.
+4. Observe console errors and page errors before navigating production Chromium
+   to the same primary URL. Forward exactly one existing browser list revalidation
+   GET to the real API and hold its response at a promise barrier. While held,
+   require the hydrated DOM to retain the identical authoritative Monitor and
+   rows82,81; it must not clear SSR content for a loading placeholder. Release,
+   settle the response, and retain the same rows. Require zero hydration errors
+   and page errors; record any other console errors without raw message bodies.
+5. Use only keyboard input for the create/detail workflow. Tab from the document
+   to Name (maximum10 Tabs), type `Monitor B`, tab through URL and interval and
+   enter the same `/ok` URL and60 seconds, retain enabled=true, then Tab/Enter to
+   submit. Require one real201 and one visible/durable Monitor B. Tab to B's
+   Show history button (maximum20 Tabs) and press Enter; verify its detail/history
+   URL and empty history. No pointer click or programmatic focus in this workflow.
+6. Run one axe scan on each of login, Monitor list (A and B), and selected A
+   detail/history (rows82,81), with fixed data. Also check one main landmark,
+   one h1 and no skipped heading level on each screen. Serious/critical violations
+   must be zero. Record every lower-severity violation and incomplete result by
+   rule ID, impact and count without raw DOM/HTML. Do not add scans to seek a
+   better result; after a concrete fix rerun only necessary authoring coverage.
+
+Existing E06 client revalidation may remain; initial hydration must use the same
+SSR payload, and privacy is checked before revalidation. Reuse the existing error
+interception assertions; change only fixture setup or observation made necessary
+by the SSR boundary, never their product/error expectations.
+
+## Scope, budget and cleanup
+
+Only E08 SSR/hydration/keyboard/accessibility and required profile metadata.
+No backend/business changes, new authentication framework, design system,
+generic observer framework, performance tuning, E09 or phase-2 work.
+One baseline; load runs0, automatic retries0, parameter sweeps0. Record every
+invocation/failure. Root owns the independent final affected regression gate.
+Preserve all old evidence, migrations and runtime pins. New evidence is confined
+to `evidence/phase-1/E08`. Refuse occupied listeners, settle held routes, close
+owned contexts/processes and drop only disposable schemas; retain public data
+and the PostgreSQL volume. No browser capture/storage-state files or push.
diff --git a/playwright.config.ts b/playwright.config.ts
index 9b818d9..54560a9 100644
--- a/playwright.config.ts
+++ b/playwright.config.ts
@@ -20,6 +20,6 @@ export default defineConfig({
   webServer: [
     { command: 'node scripts/fixture.mjs', url: 'http://127.0.0.1:4321/ok', reuseExistingServer: false },
     { command: 'node scripts/test-api.mjs', url: 'http://127.0.0.1:4322/api/session', reuseExistingServer: false },
-    { command: 'npm run dev', url: 'http://127.0.0.1:4323/monitors', reuseExistingServer: false, timeout: 90_000 },
+    { command: 'npm run start', url: 'http://127.0.0.1:4323/monitors', reuseExistingServer: false, timeout: 90_000 },
   ],
 });
diff --git a/tests/browser/rendering.spec.ts b/tests/browser/rendering.spec.ts
new file mode 100644
index 0000000..bfd03b3
--- /dev/null
+++ b/tests/browser/rendering.spec.ts
@@ -0,0 +1,53 @@
+import { createHash } from 'node:crypto';
+import { execFileSync } from 'node:child_process';
+import { mkdirSync, readFileSync, writeFileSync } from 'node:fs';
+import { test, expect, csrfHeaders, safeRequest } from './authenticated';
+
+const baseline = process.env.E08_BASELINE === '1';
+const runIds = ['00000000-0000-4000-8000-000000000082', '00000000-0000-4000-8000-000000000081'];
+
+test('fixed authenticated SSR, hydration, keyboard and accessibility boundary', async ({ page }) => {
+  const evidence: Record<string, unknown> = {
+    start: '50f633fe194c079db0cc438cac1a517702aef234',
+    fixtureSha256: createHash('sha256').update(readFileSync('evidence/phase-1/E08/fixtures.md')).digest('hex'),
+    production: true, baseline, completed: [] as string[],
+  };
+  const existing = await safeRequest(() => page.request.get('/api/monitors'));
+  expect(existing.status()).toBe(200);
+  for (const row of (await existing.json()).data) {
+    const headers = await csrfHeaders(page.request);
+    expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
+  }
+  const headers = await csrfHeaders(page.request);
+  const created = await safeRequest(() => page.request.post('/api/monitors', { headers,
+    data: { name: 'Monitor A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true } }));
+  expect(created.status()).toBe(201);
+  const monitor = (await created.json()).data.monitor.id as string;
+  expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
+  execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+    '--set', 'ON_ERROR_STOP=1', '--command', `INSERT INTO e04_browser.check_runs VALUES
+      ('${runIds[1]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:00.000Z','2026-08-28T00:00:00.000Z'),
+      ('${runIds[0]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:01.000Z','2026-08-28T00:00:01.000Z')`],
+  { stdio: 'pipe' });
+  const primary = `/monitors?history=${monitor}&limit=3`;
+  try {
+    const initial = await safeRequest(() => page.request.get(primary));
+    expect(initial.status()).toBe(200);
+    const html = await initial.text(); // Kept in memory; never written or included in an assertion failure.
+    const observed = {
+      status: initial.status(), monitorRendered: html.includes('aria-label="Monitor A"'),
+      renderedHistoryIds: [...html.matchAll(/<tr[^>]*data-check-id="([^"]+)"/g)].map(match => match[1]),
+      mainCount: [...html.matchAll(/<main[ >]/g)].length, h1Count: [...html.matchAll(/<h1[ >]/g)].length,
+      beforeBrowserJavaScript: true,
+    };
+    evidence.initialHtml = observed;
+    expect(observed.monitorRendered, 'Authenticated initial HTML must render Monitor A before JavaScript').toBe(true);
+    expect(observed.renderedHistoryIds).toEqual(runIds);
+    expect(observed.mainCount).toBe(1);
+    expect(observed.h1Count).toBe(1);
+  } finally {
+    mkdirSync('output/phase-1/e08', { recursive: true });
+    writeFileSync(`output/phase-1/e08/${baseline ? 'baseline' : 'rendering'}.json`, `${JSON.stringify(evidence, null, 2)}\n`);
+  }
+});


