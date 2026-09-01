## `test(rendering): verify privacy keyboard and accessibility`

diff --git a/evidence/phase-1/E08/affected-browser.json b/evidence/phase-1/E08/affected-browser.json
new file mode 100644
index 0000000..dece0e0
--- /dev/null
+++ b/evidence/phase-1/E08/affected-browser.json
@@ -0,0 +1,93 @@
+{
+  "stats": {
+    "startTime": "2026-08-28T03:56:17.724Z",
+    "duration": 12847.151,
+    "expected": 8,
+    "skipped": 0,
+    "unexpected": 0,
+    "flaky": 0
+  },
+  "tests": [
+    {
+      "file": "errors.spec.ts",
+      "title": "input error category does not depend on prose: Arbitrary server prose A",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "input error category does not depend on prose: Different server prose B",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "a missing Monitor is an API error, not a completed failed check",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "list failures use the internal error category",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "a malformed error envelope uses a safe category without a successful mutation",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "network failure cannot look like a successful mutation",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "errors.spec.ts",
+      "title": "a malformed success payload is rejected before rendering a Monitor",
+      "passed": true,
+      "attempts": 1
+    },
+    {
+      "file": "history.spec.ts",
+      "title": "fixed tied history pages restore URL state and ignore the held older filter response",
+      "passed": true,
+      "attempts": 1
+    }
+  ],
+  "history": {
+    "fixtureSha256": "a22129aa5b5137b1b4da6907c16f80e3193b58449c91a145ec1329a9d9233bd1",
+    "completed": [
+      "actual browser cursor header, seven unique originals across pages with newer eighth insertion",
+      "cursor page back/forward/reload",
+      "FAILED filter clears cursor, preserves size and displays exactly three failed rows",
+      "newer All result displayed while one real SUCCEEDED response remains held",
+      "released older response cannot overwrite current rows, URL or pending/error status",
+      "filter back/forward/reload"
+    ],
+    "result": "PASS",
+    "heldRealReads": 1,
+    "originalContinuation": [
+      7,
+      6,
+      5,
+      4,
+      3,
+      2,
+      1
+    ],
+    "currentAll": [
+      8,
+      7,
+      6
+    ],
+    "staleSucceeded": [
+      8,
+      7,
+      5
+    ],
+    "heldRoutesSettled": true
+  }
+}
diff --git a/evidence/phase-1/E08/cleanup.json b/evidence/phase-1/E08/cleanup.json
new file mode 100644
index 0000000..1c86202
--- /dev/null
+++ b/evidence/phase-1/E08/cleanup.json
@@ -0,0 +1,10 @@
+{
+  "checkedAt": "2026-08-28T04:01:22.080Z",
+  "schemas": [
+    "public"
+  ],
+  "listenersOn4321Through4325And4999": [],
+  "ownedBrowserSchemaDropped": true,
+  "publicAndVolumeNotTargetedForRemoval": true,
+  "heldRoutesSettled": true
+}
diff --git a/evidence/phase-1/E08/invocations.jsonl b/evidence/phase-1/E08/invocations.jsonl
new file mode 100644
index 0000000..d7327cf
--- /dev/null
+++ b/evidence/phase-1/E08/invocations.jsonl
@@ -0,0 +1,12 @@
+{"name":"database-up","command":"node scripts/database.mjs up","startedAt":"2026-08-28T03:43:42.810Z","elapsedSeconds":1.078,"exitCode":0}
+{"name":"baseline","command":"npm run test:e2e -- tests/browser/rendering.spec.ts","startedAt":"2026-08-28T03:43:43.890Z","elapsedSeconds":13.56,"exitCode":1}
+{"name":"baseline-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T03:43:57.451Z","elapsedSeconds":0.189,"exitCode":0}
+{"name":"axe-install","command":"npm install --save-dev --save-exact --ignore-scripts --no-audit --no-fund axe-core@4.11.0","startedAt":"2026-08-28T03:46:48.556Z","elapsedSeconds":0.695,"exitCode":0}
+{"name":"typecheck-1","command":"npm run typecheck","startedAt":"2026-08-28T03:54:14.301Z","elapsedSeconds":2.394,"exitCode":0}
+{"name":"build-1","command":"npm run build","startedAt":"2026-08-28T03:54:16.697Z","elapsedSeconds":9.834,"exitCode":0}
+{"name":"rendering-1","command":"npm run test:e2e -- tests/browser/rendering.spec.ts","startedAt":"2026-08-28T03:54:26.531Z","elapsedSeconds":14.488,"exitCode":0}
+{"name":"rendering-1-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T03:54:41.019Z","elapsedSeconds":0.187,"exitCode":0}
+{"name":"affected-browser-1","command":"npm run test:e2e -- tests/browser/errors.spec.ts tests/browser/history.spec.ts","startedAt":"2026-08-28T03:56:17.178Z","elapsedSeconds":13.408,"exitCode":0}
+{"name":"affected-browser-1-cleanup","command":"node scripts/database.mjs drop e04_browser","startedAt":"2026-08-28T03:56:30.588Z","elapsedSeconds":0.169,"exitCode":0}
+{"name":"final-database-inspection","command":"docker compose --project-name wse-industry --file compose.yaml exec --no-TTY postgres psql --username wse_industry --dbname monitor --tuples-only --no-align --command <read-only non-system schema query>","startedAt":"2026-08-28T04:01:22.080Z","elapsedSeconds":0.136889625,"exitCode":0,"result":"public only"}
+{"name":"final-listener-inspection","command":"lsof -nP -iTCP:4321-4325 -iTCP:4999 -sTCP:LISTEN","startedAt":"2026-08-28T04:01:22.080Z","elapsedSeconds":0.000005625,"exitCode":1,"result":"no matching listeners (expected lsof exit 1)"}
diff --git a/evidence/phase-1/E08/rendering.json b/evidence/phase-1/E08/rendering.json
new file mode 100644
index 0000000..015b54b
--- /dev/null
+++ b/evidence/phase-1/E08/rendering.json
@@ -0,0 +1,97 @@
+{
+  "start": "50f633fe194c079db0cc438cac1a517702aef234",
+  "fixtureSha256": "77beb223b83fc659830bf090eb5c3a469f7f14bce5dc3e12275587d3a89b2542",
+  "production": true,
+  "baseline": false,
+  "completed": [
+    "Alice server HTML includes authoritative Monitor/latest/history before JavaScript",
+    "same URL Alice/Bob/anonymous privacy and credential-free HTML with embedded RSC/client props",
+    "first client state matches SSR while the one real revalidation is held, then remains stable",
+    "keyboard-only create B and selected detail/history navigation",
+    "one default-rule axe scan each for login/list/detail; no serious or critical violations"
+  ],
+  "initialHtml": {
+    "status": 200,
+    "monitorRendered": true,
+    "renderedHistoryIds": [
+      "00000000-0000-4000-8000-000000000082",
+      "00000000-0000-4000-8000-000000000081"
+    ],
+    "mainCount": 1,
+    "h1Count": 1,
+    "beforeBrowserJavaScript": true
+  },
+  "privacy": {
+    "sameUrl": true,
+    "bobStatus": 200,
+    "anonymousStatus": 307,
+    "foreignRecordsAbsent": true,
+    "privateNoStore": true,
+    "runtimeSecretsAbsentFromHtmlAndEmbeddedRsc": true,
+    "standaloneRscRequestChecked": false
+  },
+  "hydration": {
+    "sameInitialHistoryIds": [
+      "00000000-0000-4000-8000-000000000082",
+      "00000000-0000-4000-8000-000000000081"
+    ],
+    "heldRealRevalidations": 1,
+    "initialRowsRetained": true
+  },
+  "keyboard": {
+    "nameTabs": 2,
+    "detailTabs": 10,
+    "createPosts": 1,
+    "durableB": 1,
+    "detailReached": true,
+    "pointerOrProgrammaticFocus": false
+  },
+  "result": "PASS",
+  "heldRoutesSettled": true,
+  "errors": {
+    "hydration": 0,
+    "page": 0,
+    "otherConsole": 0
+  },
+  "accessibility": [
+    {
+      "screen": "login",
+      "headingLevels": [
+        1
+      ],
+      "version": "4.11.0",
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 32
+    },
+    {
+      "screen": "list",
+      "headingLevels": [
+        1,
+        2,
+        2,
+        3,
+        3
+      ],
+      "version": "4.11.0",
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 33
+    },
+    {
+      "screen": "detail",
+      "headingLevels": [
+        1,
+        2,
+        2,
+        3,
+        4,
+        3
+      ],
+      "version": "4.11.0",
+      "violations": [],
+      "incomplete": [],
+      "passedRules": 38
+    }
+  ]
+}
diff --git a/evidence/phase-1/E08/verification.md b/evidence/phase-1/E08/verification.md
new file mode 100644
index 0000000..9bf3d37
--- /dev/null
+++ b/evidence/phase-1/E08/verification.md
@@ -0,0 +1,102 @@
+# Phase-1 E08 author verification
+
+Thread E08, attempt1, branch `track/industry-spring`.
+START `50f633fe194c079db0cc438cac1a517702aef234`.
+SPEC_REVISION `2ada57a71cd34fa2fae9809415c362a8bbfcdf02`.
+Frozen fixture SHA256:
+`77beb223b83fc659830bf090eb5c3a469f7f14bce5dc3e12275587d3a89b2542`.
+
+## Change and boundary
+
+The Monitor page loads the current request's authenticated collection and selected
+history on the server, with `force-dynamic` and `cache: 'no-store'`. The server-only
+loader forwards the session cookie to the existing fixed backend origin; the
+client receives only decoded records, loading/error state and selected history.
+The same payload renders HTML and initializes the client cache. Existing list
+revalidation remains and does not clear those rows. Static Monitor/login shells
+are server components; forms and mutation controls remain client components.
+No backend, migration, business rule, security filter or CSS changes were needed.
+
+The only added package is dev-only, exact `axe-core@4.11.0`. Previous dependency
+versions remain unchanged. Browser checks now use the existing production start
+command; build first when invoking the browser suite separately.
+
+## Executions and results
+
+Commands used Node24.19.0 through `fnm exec --using 24.19.0`; production Next and
+pinned Chromium revision1234, one worker, retries0. `invocations.jsonl` records
+the timed commands, setup and cleanup. A pre-freeze read-only
+`npm view axe-core@4.11.0 version` returned4.11.0; its timing was not retained.
+
+| Execution | Result | Command elapsed |
+| --- | --- | --- |
+| One unchanged-product baseline, rendering case | Expected failure: HTML200 but no Monitor article/history; main1, h1=0 | 13.560s |
+| Typecheck | PASS | 2.394s |
+| Production build | PASS | 9.834s |
+| Fixed E08 rendering case | PASS, one test | 14.488s |
+| Existing error/history browser regression | PASS, eight tests; report duration12.847151s | 13.408s |
+
+The baseline used the already root-verified E07 production artifact; no product
+edit preceded it. It stopped at the missing server-rendered Monitor, so no
+baseline hydration, keyboard or accessibility result is claimed. The metadata
+transition and frozen fixture were then committed together. No post-change
+product test failed; no automatic retry, load run or parameter sweep occurred.
+Author full `npm run verify` invocations:0. Backend bytes are unchanged from START;
+root's E07 Java38 verification is reused, not represented as a fresh E08 Java run.
+Root independently runs the final affected gate after these author commits.
+
+The first metadata-containing patch was rejected before execution. The agent
+did not retry that marker change through another tool. Root obtained narrow
+approval and supplied only SPEC_REVISION/TRACK metadata edits for the first E08
+commit. One later move/add patch syntax attempt also failed before execution and
+was corrected. Neither was a product-test failure or a baseline repetition.
+
+## Fixed-case observations
+
+`rendering.json` contains sanitized observations, not response bodies or secrets.
+Alice's initial HTML200 included Monitor A, latest82 and history82,81. Bob's exact
+same URL returned200 without Alice's records; anonymous access redirected307 to
+login. HTML was private/no-store. Runtime password, session-cookie and known
+CSRF values were absent from the HTML bodies, including their embedded RSC/client
+props stream. **No standalone RSC request was sent or checked.** The original
+observation label was clarified after the passing run to
+`runtimeSecretsAbsentFromHtmlAndEmbeddedRsc`, with
+`standaloneRscRequestChecked: false`; requests and assertions were not changed.
+
+While one real list revalidation response was held, first hydrated content kept
+the identical Monitor and history82,81. Release settled the route and retained
+the rows. Hydration errors0, page errors0, other console errors0. Keyboard-only
+creation used2 Tabs to Name, emitted one real201 POST and produced one durable
+Monitor B. Ten Tabs reached B's detail control; Enter selected its history URL.
+There was no pointer input or programmatic focus in this workflow.
+
+Exactly three default-rule axe4.11.0 scans ran: login/list/detail once each.
+Each had one main, one h1 and no skipped heading level. All violations, including
+serious/critical, were0; incompletes0. Passed rules were32/33/38 respectively.
+No exclusions, disabled rules or extra scans were used.
+
+## Preserved E07 regression and observation adapter
+
+Selected history is now present in server HTML. With root approval,
+`history.spec.ts` initial navigation and its two reload observations use the
+actual document response200 instead of waiting for an initial browser API GET
+that SSR makes unnecessary. The same URL/rows remain asserted. The first next
+cursor is read from the actual next-page URL; the following real API response's
+`X-Next-Cursor` is still consumed and matched against the third-page URL.
+Finite limit, original seven/eighth IDs, filters, back/forward/reload and the held
+older response conditions are unchanged. No synthetic response or extra product
+fetch was added. `affected-browser.json` records the eight passes and exact
+seven-original continuation, newer All rows and held stale-result observations.
+All seven existing error cases and all committed evidence/E07 files are unchanged.
+
+## Cleanup
+
+Each browser invocation stopped its owned services and dropped only the disposable
+`e04_browser` schema. Final read-only inspection found only `public` among
+non-system schemas and no listeners on4321–4325 or4999. Held routes were settled;
+owned browser contexts were closed. The public schema and PostgreSQL volume were
+not targeted for removal. `cleanup.json` records the observation. No private HTML,
+cookie/token material, browser captures or storage-state files were committed.
+
+No E09, phase-2, E25, main/spec/index/tag edits or push. Author work is complete;
+root verification and any next explicit dispatch are separate.
diff --git a/package-lock.json b/package-lock.json
index f1908cb..3db1833 100644
--- a/package-lock.json
+++ b/package-lock.json
@@ -17,6 +17,7 @@
         "@types/node": "24.10.1",
         "@types/react": "19.2.18",
         "@types/react-dom": "19.2.5",
+        "axe-core": "4.11.0",
         "typescript": "5.9.3"
       },
       "engines": {
@@ -784,6 +785,16 @@
         "@types/react": "^19.2.0"
       }
     },
+    "node_modules/axe-core": {
+      "version": "4.11.0",
+      "resolved": "https://registry.npmjs.org/axe-core/-/axe-core-4.11.0.tgz",
+      "integrity": "sha512-ilYanEU8vxxBexpJd8cWM4ElSQq4QctCLKih0TSfjIfCQTeyH/6zVrmIJfLPrKTKJRbiG+cfnZbQIjAlJmF1jQ==",
+      "dev": true,
+      "license": "MPL-2.0",
+      "engines": {
+        "node": ">=4"
+      }
+    },
     "node_modules/baseline-browser-mapping": {
       "version": "2.11.19",
       "resolved": "https://registry.npmjs.org/baseline-browser-mapping/-/baseline-browser-mapping-2.11.19.tgz",
diff --git a/package.json b/package.json
index 3bf1803..af29b77 100644
--- a/package.json
+++ b/package.json
@@ -30,6 +30,7 @@
     "@types/node": "24.10.1",
     "@types/react": "19.2.18",
     "@types/react-dom": "19.2.5",
+    "axe-core": "4.11.0",
     "typescript": "5.9.3"
   }
 }
diff --git a/tests/browser/rendering.spec.ts b/tests/browser/rendering.spec.ts
index bfd03b3..5265c03 100644
--- a/tests/browser/rendering.spec.ts
+++ b/tests/browser/rendering.spec.ts
@@ -1,40 +1,71 @@
 import { createHash } from 'node:crypto';
 import { execFileSync } from 'node:child_process';
 import { mkdirSync, readFileSync, writeFileSync } from 'node:fs';
-import { test, expect, csrfHeaders, safeRequest } from './authenticated';
+import axe from 'axe-core';
+import type { APIRequestContext, BrowserContext, Locator, Page, Route } from '@playwright/test';
+import { test, expect, csrfHeaders, fixturePassword, safeRequest } from './authenticated';
 
 const baseline = process.env.E08_BASELINE === '1';
 const runIds = ['00000000-0000-4000-8000-000000000082', '00000000-0000-4000-8000-000000000081'];
 
-test('fixed authenticated SSR, hydration, keyboard and accessibility boundary', async ({ page }) => {
+async function clearOwned(request: APIRequestContext) {
+  const response = await safeRequest(() => request.get('/api/monitors'));
+  expect(response.status()).toBe(200);
+  for (const row of (await response.json()).data) {
+    const headers = await csrfHeaders(request);
+    expect((await safeRequest(() => request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
+  }
+}
+
+async function tabTo(page: Page, target: Locator, maximum: number) {
+  for (let count = 0; count <= maximum; count++) {
+    if (await target.evaluate(element => document.activeElement === element)) return count;
+    if (count < maximum) await page.keyboard.press('Tab');
+  }
+  throw new Error('Keyboard target was not reachable within the frozen Tab bound');
+}
+
+test('fixed authenticated SSR, hydration, keyboard and accessibility boundary', async ({ page, browser, context }) => {
   const evidence: Record<string, unknown> = {
     start: '50f633fe194c079db0cc438cac1a517702aef234',
     fixtureSha256: createHash('sha256').update(readFileSync('evidence/phase-1/E08/fixtures.md')).digest('hex'),
     production: true, baseline, completed: [] as string[],
   };
-  const existing = await safeRequest(() => page.request.get('/api/monitors'));
-  expect(existing.status()).toBe(200);
-  for (const row of (await existing.json()).data) {
-    const headers = await csrfHeaders(page.request);
-    expect((await safeRequest(() => page.request.delete(`/api/monitors/${row.monitor.id}`, { headers }))).status()).toBe(200);
-  }
-  const headers = await csrfHeaders(page.request);
-  const created = await safeRequest(() => page.request.post('/api/monitors', { headers,
-    data: { name: 'Monitor A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true } }));
-  expect(created.status()).toBe(201);
-  const monitor = (await created.json()).data.monitor.id as string;
-  expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
-  execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
-    'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
-    '--set', 'ON_ERROR_STOP=1', '--command', `INSERT INTO e04_browser.check_runs VALUES
-      ('${runIds[1]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:00.000Z','2026-08-28T00:00:00.000Z'),
-      ('${runIds[0]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:01.000Z','2026-08-28T00:00:01.000Z')`],
-  { stdio: 'pipe' });
-  const primary = `/monitors?history=${monitor}&limit=3`;
+  const completed = evidence.completed as string[];
+  const scans: unknown[] = [];
+  const errors = { hydration: 0, page: 0, otherConsole: 0 };
+  const held = Promise.withResolvers<void>();
+  void held.promise.catch(() => {});
+  const release = Promise.withResolvers<void>();
+  const routes: Promise<void>[] = [];
+  let heldReads = 0;
+  let bob: BrowserContext | undefined;
+  let anonymous: BrowserContext | undefined;
+  let intercept: ((route: Route) => Promise<void>) | undefined;
   try {
+    bob = await browser.newContext({ baseURL: 'http://127.0.0.1:4323' });
+    anonymous = await browser.newContext({ baseURL: 'http://127.0.0.1:4323' });
+    const bobHeaders = await csrfHeaders(bob.request);
+    expect((await safeRequest(() => bob!.request.post('/api/session/login', { headers: bobHeaders,
+      data: { username: 'bob-e04', password: fixturePassword('bob') } }))).status()).toBe(200);
+    await clearOwned(page.request);
+    await clearOwned(bob.request);
+    const headers = await csrfHeaders(page.request);
+    const created = await safeRequest(() => page.request.post('/api/monitors', { headers,
+      data: { name: 'Monitor A', url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true } }));
+    expect(created.status()).toBe(201);
+    const monitor = (await created.json()).data.monitor.id as string;
+    expect(monitor).toMatch(/^[a-f0-9-]{36}$/);
+    execFileSync('docker', ['compose', '--project-name', 'wse-industry', '--file', 'compose.yaml',
+      'exec', '--no-TTY', 'postgres', 'psql', '--username', 'wse_industry', '--dbname', 'monitor',
+      '--set', 'ON_ERROR_STOP=1', '--command', `INSERT INTO e04_browser.check_runs VALUES
+        ('${runIds[1]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:00.000Z','2026-08-28T00:00:00.000Z'),
+        ('${runIds[0]}','${monitor}','MANUAL','SUCCEEDED',200,1,null,'2026-08-28T00:00:01.000Z','2026-08-28T00:00:01.000Z')`],
+    { stdio: 'pipe' });
+    const primary = `/monitors?history=${monitor}&limit=3`;
     const initial = await safeRequest(() => page.request.get(primary));
     expect(initial.status()).toBe(200);
-    const html = await initial.text(); // Kept in memory; never written or included in an assertion failure.
+    const html = await initial.text(); // Never written or included in an assertion failure.
     const observed = {
       status: initial.status(), monitorRendered: html.includes('aria-label="Monitor A"'),
       renderedHistoryIds: [...html.matchAll(/<tr[^>]*data-check-id="([^"]+)"/g)].map(match => match[1]),
@@ -46,7 +77,148 @@ test('fixed authenticated SSR, hydration, keyboard and accessibility boundary',
     expect(observed.renderedHistoryIds).toEqual(runIds);
     expect(observed.mainCount).toBe(1);
     expect(observed.h1Count).toBe(1);
+    if (baseline) return;
+    expect(html.includes(`data-latest-check-id="${runIds[0]}"`)).toBe(true);
+    expect(initial.headers()['cache-control']).toContain('private');
+    expect(initial.headers()['cache-control']).toContain('no-store');
+    completed.push('Alice server HTML includes authoritative Monitor/latest/history before JavaScript');
+
+    const bobHtml = await safeRequest(() => bob!.request.get(primary));
+    expect(bobHtml.status()).toBe(200);
+    const anonymousHtml = await safeRequest(() => anonymous!.request.get(primary, { maxRedirects: 0 }));
+    if (anonymousHtml.status() !== 401) {
+      expect([302, 303, 307, 308]).toContain(anonymousHtml.status());
+      expect(new URL(anonymousHtml.headers().location, 'http://127.0.0.1:4323').pathname).toBe('/login');
+    }
+    const otherBodies = [await bobHtml.text(), await anonymousHtml.text()];
+    for (const body of otherBodies) {
+      expect(body.includes('Monitor A') || runIds.some(id => body.includes(id))).toBe(false);
+    }
+    const secrets = [fixturePassword('alice'), fixturePassword('bob'), headers['X-CSRF-TOKEN'], bobHeaders['X-CSRF-TOKEN'],
+      ...(await context.cookies()).map(cookie => cookie.value), ...(await bob.cookies()).map(cookie => cookie.value)];
+    for (const body of [html, ...otherBodies]) expect(secrets.every(secret => !body.includes(secret))).toBe(true);
+    evidence.privacy = { sameUrl: true, bobStatus: bobHtml.status(), anonymousStatus: anonymousHtml.status(),
+      foreignRecordsAbsent: true, runtimeSecretsAbsentFromHtmlAndEmbeddedRsc: true, privateNoStore: true,
+      standaloneRscRequestChecked: false };
+    completed.push('same URL Alice/Bob/anonymous privacy and credential-free HTML with embedded RSC/client props');
+
+    page.on('console', message => {
+      if (/hydrat|did not match|server rendered html/i.test(message.text())) errors.hydration++;
+      else if (message.type() === 'error') errors.otherConsole++;
+    });
+    page.on('pageerror', () => { errors.page++; });
+    let createPosts = 0;
+    page.on('request', request => {
+      if (request.method() === 'POST' && new URL(request.url()).pathname === '/api/monitors') createPosts++;
+    });
+    intercept = route => {
+      if (route.request().method() !== 'GET' || heldReads !== 0) return route.continue();
+      heldReads++;
+      const work = (async () => {
+        const real = await safeRequest(() => route.fetch());
+        expect(real.status()).toBe(200);
+        held.resolve();
+        await release.promise;
+        await route.fulfill({ response: real });
+      })();
+      routes.push(work);
+      void work.catch(error => held.reject(error));
+      return work;
+    };
+    await page.route('**/api/monitors', intercept);
+    await page.goto(primary);
+    await held.promise;
+    const a = page.getByRole('article', { name: 'Monitor A', exact: true });
+    const matchingInitial = async () => {
+      await expect(a).toBeVisible();
+      await expect(a.getByText('60s interval · Enabled', { exact: true })).toBeVisible();
+      await expect(a.locator('[data-latest-check-id]')).toHaveAttribute('data-latest-check-id', runIds[0]);
+      await expect(a.locator('tbody tr')).toHaveCount(2);
+      for (const [index, id] of runIds.entries()) await expect(a.locator('tbody tr').nth(index)).toHaveAttribute('data-check-id', id);
+      await expect(page.getByText('Loading monitors…', { exact: true })).toHaveCount(0);
+    };
+    await matchingInitial();
+    expect(errors.hydration).toBe(0);
+    expect(errors.page).toBe(0);
+    const delivered = page.waitForResponse(response => new URL(response.url()).pathname === '/api/monitors'
+      && response.request().method() === 'GET');
+    release.resolve();
+    expect((await delivered).status()).toBe(200);
+    await (await delivered).finished();
+    await Promise.all(routes);
+    await matchingInitial();
+    expect(heldReads).toBe(1);
+    evidence.hydration = { sameInitialHistoryIds: runIds, heldRealRevalidations: heldReads, initialRowsRetained: true };
+    completed.push('first client state matches SSR while the one real revalidation is held, then remains stable');
+
+    const nameTabs = await tabTo(page, page.getByLabel('Name', { exact: true }), 10);
+    await page.keyboard.press('ControlOrMeta+A');
+    await page.keyboard.type('Monitor B');
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('URL', { exact: true })).toBeFocused();
+    await page.keyboard.press('ControlOrMeta+A');
+    await page.keyboard.type('http://127.0.0.1:4321/ok');
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('Interval (seconds)', { exact: true })).toBeFocused();
+    await page.keyboard.press('ControlOrMeta+A');
+    await page.keyboard.type('60');
+    await page.keyboard.press('Tab');
+    await expect(page.getByLabel('Enabled', { exact: true })).toBeFocused();
+    await expect(page.getByLabel('Enabled', { exact: true })).toBeChecked();
+    await page.keyboard.press('Tab');
+    await expect(page.getByRole('button', { name: 'Create monitor', exact: true })).toBeFocused();
+    const [createdB] = await Promise.all([page.waitForResponse(response => new URL(response.url()).pathname === '/api/monitors'
+      && response.request().method() === 'POST'), page.keyboard.press('Enter')]);
+    expect(createdB.status()).toBe(201);
+    const bId = (await createdB.json()).data.monitor.id as string;
+    const b = page.getByRole('article', { name: 'Monitor B', exact: true });
+    await expect(b).toHaveCount(1);
+    const durable = await safeRequest(() => page.request.get('/api/monitors'));
+    expect(durable.status()).toBe(200);
+    expect((await durable.json()).data.filter((row: { monitor: { name: string } }) => row.monitor.name === 'Monitor B')).toHaveLength(1);
+    expect(createPosts).toBe(1);
+    const detailTabs = await tabTo(page, b.getByRole('button', { name: 'Show history', exact: true }), 20);
+    await page.keyboard.press('Enter');
+    await expect(page).toHaveURL(url => url.searchParams.get('history') === bId && url.searchParams.get('limit') === '3');
+    await expect(b.getByText('No historical checks.', { exact: true })).toBeVisible();
+    evidence.keyboard = { nameTabs, detailTabs, createPosts, durableB: 1, detailReached: true, pointerOrProgrammaticFocus: false };
+    completed.push('keyboard-only create B and selected detail/history navigation');
+
+    for (const [screen, url] of [['login', '/login'], ['list', '/monitors'], ['detail', primary]] as const) {
+      await page.goto(url);
+      await expect(page.getByRole('main')).toHaveCount(1);
+      await expect(page.getByRole('heading', { level: 1 })).toHaveCount(1);
+      if (screen !== 'login') await expect(page.getByRole('article')).toHaveCount(2);
+      if (screen === 'detail') await matchingInitial();
+      const levels = await page.locator('main h1, main h2, main h3, main h4, main h5, main h6')
+        .evaluateAll(headings => headings.map(heading => Number(heading.tagName.slice(1))));
+      expect(levels[0]).toBe(1);
+      expect(levels.every((level, index) => index === 0 || level <= levels[index - 1] + 1)).toBe(true);
+      await page.addScriptTag({ content: axe.source });
+      const result = await page.evaluate(async () => {
+        const scanner = (window as typeof window & { axe: typeof import('axe-core') }).axe;
+        const result = await scanner.run();
+        const summary = (rows: typeof result.violations) => rows.map(row => ({ id: row.id, impact: row.impact, nodes: row.nodes.length }));
+        return { version: scanner.version, violations: summary(result.violations), incomplete: summary(result.incomplete),
+          passedRules: result.passes.length };
+      });
+      scans.push({ screen, headingLevels: levels, ...result });
+      expect(result.version).toBe('4.11.0');
+      expect(result.violations.filter(row => row.impact === 'serious' || row.impact === 'critical')).toEqual([]);
+    }
+    expect(errors.hydration).toBe(0);
+    expect(errors.page).toBe(0);
+    completed.push('one default-rule axe scan each for login/list/detail; no serious or critical violations');
+    evidence.result = 'PASS';
   } finally {
+    release.resolve();
+    const settled = await Promise.allSettled(routes);
+    if (intercept) await page.unroute('**/api/monitors', intercept);
+    await bob?.close();
+    await anonymous?.close();
+    evidence.heldRoutesSettled = settled.every(result => result.status === 'fulfilled');
+    evidence.errors = errors;
+    evidence.accessibility = scans;
     mkdirSync('output/phase-1/e08', { recursive: true });
     writeFileSync(`output/phase-1/e08/${baseline ? 'baseline' : 'rendering'}.json`, `${JSON.stringify(evidence, null, 2)}\n`);
   }
