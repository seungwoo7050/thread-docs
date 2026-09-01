## `test(http): verify browser error categories with fixed fixtures`

diff --git a/evidence/E02/runs.jsonl b/evidence/E02/runs.jsonl
new file mode 100644
index 0000000..f1b8cf2
--- /dev/null
+++ b/evidence/E02/runs.jsonl
@@ -0,0 +1,6 @@
+{"phase":"baseline","command":"fnm exec --using=24.19.0 npm run test:api -- '-Dtest=MonitorFunctionalTest#rejectsBlankNameAtRuntime'","startedAt":"2026-08-27T23:23:40.326Z","elapsedSeconds":11.338,"clock":"Maven total time","exitCode":1,"tests":1,"failures":1,"errors":0,"outcome":"Expected 400 INVALID_INPUT; observed 201 CREATED with a whitespace-name Monitor. Stopped after one counterexample."}
+{"phase":"post-fix-first","command":"mvn -B -ntp -f backend/pom.xml package","startedAt":"2026-08-27T23:34:02.158Z","elapsedSeconds":8.251,"exitCode":0,"tests":17,"failures":0}
+{"phase":"post-fix-first","command":"npm run typecheck","startedAt":"2026-08-27T23:34:10.409Z","elapsedSeconds":1.312,"exitCode":0}
+{"phase":"post-fix-first","command":"npm run build","startedAt":"2026-08-27T23:34:11.721Z","elapsedSeconds":7.43,"exitCode":0}
+{"phase":"post-fix-first","command":"npm run test:e2e","startedAt":"2026-08-27T23:34:19.151Z","elapsedSeconds":11.599,"exitCode":1,"tests":9,"passed":2,"failed":7,"outcome":"All seven new tests had ambiguous role=alert locators (application alert plus Next route announcer); unchanged E01 browser tests passed."}
+{"phase":"post-fix-browser-locator","command":"/usr/bin/time -p fnm exec --using=24.19.0 npm run test:e2e","startedAt":"2026-08-27T23:36:16.659Z","elapsedSeconds":8.7,"clock":"/usr/bin/time wall","exitCode":0,"tests":9,"passed":9,"failed":0,"outcome":"Identical fixtures and expectations; locator restricted to the application main landmark. No product changes since the passing Java/type/build gates."}
diff --git a/evidence/E02/verification.md b/evidence/E02/verification.md
new file mode 100644
index 0000000..808fb12
--- /dev/null
+++ b/evidence/E02/verification.md
@@ -0,0 +1,32 @@
+# E02 verification evidence
+
+Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`; attempt 1.
+Start: `c3495c1478182bbea5bc47d78301e0bfa5275ab9`.
+Inputs and the recorded pre-execution numeric-value clarification are in `fixtures.md`.
+
+## Baseline and correction
+
+Before production edits, the single selected `MonitorFunctionalTest#rejectsBlankNameAtRuntime` posted the frozen three-space name to the real API on port 4322. Expected 400 / INVALID_INPUT; observed **201 CREATED**, `monitor.name="   "`, and an unwrapped MonitorView with `latestCheck=null`. The status assertion failed; baseline exploration stopped immediately after this one counterexample. The same test method and request now pass unchanged.
+
+The request boundary inspects JSON kinds before conversion, strips and bounds the name, accepts only integer-valued numeric intervals, requires a boolean, and checks absolute HTTP(S) syntax before the unchanged fixture-only destination policy. Spring MVC advice maps API errors to the three fixed categories. The browser checks the response envelope/payload and selects its own text by code/status, without using server prose.
+
+## Executed commands
+
+`runs.jsonl` includes every failed and successful verification command. The full runner was invoked once using `fnm exec --using=24.19.0 npm run verify`; only the affected browser suite was rerun after its test-locator correction.
+
+| Invocation | Outcome | Seconds |
+| --- | --- | ---: |
+| `npm run test:api -- '-Dtest=MonitorFunctionalTest#rejectsBlankNameAtRuntime'` via pinned fnm | Expected baseline failure: 1 test, 201 instead of 400 | 11.338 (Maven timer) |
+| Runner: `mvn -B -ntp -f backend/pom.xml package` | PASS: 4 CheckRunner tests, 1 MVC error-boundary test, 12 real-HTTP functional tests; jar built | 8.251 |
+| Runner: `npm run typecheck` | PASS | 1.312 |
+| Runner: `npm run build` | PASS | 7.430 |
+| Runner: `npm run test:e2e` | FAIL: 7 ambiguous alert locators; unchanged E01 browser cases 2/2 passed | 11.599 |
+| `/usr/bin/time -p fnm exec --using=24.19.0 npm run test:e2e` | PASS: all 9 Chromium tests | 8.70 |
+
+The initial browser failure was `getByRole('alert')` resolving both the correct application error paragraph and Next's `__next-route-announcer__`. Its snapshot showed the expected application code/text inside `main`. Only the locator changed to `getByRole('main').getByRole('alert')`; the requests, prose A/B, statuses, categories, timeout and assertions did not change. Product code was unchanged between those two browser invocations.
+
+Passing coverage includes all original packet cases; trimmed 1/100 UTF-16 boundaries; missing/null/wrong JSON kinds; numeric 60.0 accepted and string "60" rejected; malformed JSON/IDs and missing routes; exact wire envelopes, primitives and nulls; a test-injected unexpected MVC exception returning 500 without private details; and actual controlled connection-failure/timeout results preserving null httpStatus. E01's controlled 200/503, redirect and forbidden-destination tests remain. Seven browser cases inject API/transport failures; two use the real API and fixture for the original create/check flows.
+
+Budget used: one selected baseline, one full verification, two browser suite invocations total. Java suite invocations total two (baseline + full runner); typecheck one; build one. Automatic retries 0; load/benchmark/profiler runs 0; sweeps 0. No dependencies or pinned versions changed. No hosted CI execution is claimed. Acceptance passed, so no further implementation or test exploration followed.
+
+Auxiliary editing outcome: one combined evidence/locator patch was rejected for mismatched evidence context before writes; the existing content was reread and the same intended edit applied. This did not execute tests or change any fixture/expectation.
diff --git a/tests/browser/errors.spec.ts b/tests/browser/errors.spec.ts
new file mode 100644
index 0000000..ebd576d
--- /dev/null
+++ b/tests/browser/errors.spec.ts
@@ -0,0 +1,80 @@
+import { test, expect, type Page, type Route } from '@playwright/test';
+
+const inputError = 'Invalid monitor input. Check the name, URL, interval, and enabled value.';
+const internalError = 'The service could not complete the request. Try again.';
+
+async function emptyListAndCreateFailure(page: Page, create: (route: Route) => Promise<void>) {
+  await page.route('**/api/monitors', route => route.request().method() === 'GET'
+    ? route.fulfill({ json: { data: [] } }) : create(route));
+  await page.goto('/monitors');
+}
+
+async function expectFailedCreate(page: Page, code: string, message: string) {
+  await page.getByLabel('Name', { exact: true }).fill('Fixture monitor');
+  await page.getByLabel('URL', { exact: true }).fill('http://127.0.0.1:4321/ok');
+  await page.getByLabel('Interval (seconds)').fill('60');
+  await page.getByRole('button', { name: 'Create monitor' }).click();
+  await expect(page.getByRole('main').getByRole('alert')).toHaveAttribute('data-error-code', code);
+  await expect(page.getByRole('main').getByRole('alert')).toHaveText(message);
+  await expect(page.getByRole('article')).toHaveCount(0);
+  await expect(page.getByText('SUCCEEDED', { exact: true })).toHaveCount(0);
+  await expect(page.getByText('FAILED', { exact: true })).toHaveCount(0);
+  await expect(page.getByLabel('Name', { exact: true })).toHaveValue('Fixture monitor');
+  await expect(page.getByRole('button', { name: 'Create monitor' })).toBeEnabled();
+}
+
+for (const message of ['Arbitrary server prose A', 'Different server prose B']) {
+  test(`input error category does not depend on prose: ${message}`, async ({ page }) => {
+    await emptyListAndCreateFailure(page, route => route.fulfill({
+      status: 400, json: { error: { code: 'INVALID_INPUT', message } },
+    }));
+    await expectFailedCreate(page, 'INVALID_INPUT', inputError);
+    await expect(page.getByText(message, { exact: true })).toHaveCount(0);
+  });
+}
+
+test('a missing Monitor is an API error, not a completed failed check', async ({ page }) => {
+  const data = {
+    monitor: { id: '00000000-0000-4000-8000-000000000000', name: 'Fixture monitor',
+      url: 'http://127.0.0.1:4321/ok', interval: 60, enabled: true,
+      createdAt: '2026-08-28T00:00:00Z', updatedAt: '2026-08-28T00:00:00Z' },
+    latestCheck: null,
+  };
+  await page.route('**/api/monitors', route => route.fulfill({ json: { data: [data] } }));
+  await page.route('**/api/monitors/*/checks', route => route.fulfill({
+    status: 404, json: { error: { code: 'NOT_FOUND', message: 'Arbitrary server prose A' } },
+  }));
+  await page.goto('/monitors');
+  const monitor = page.getByRole('article', { name: 'Fixture monitor', exact: true });
+  await monitor.getByRole('button', { name: 'Run check' }).click();
+  await expect(page.getByRole('main').getByRole('alert')).toHaveAttribute('data-error-code', 'NOT_FOUND');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveText('Monitor not found. Reload the list and try again.');
+  await expect(monitor.getByText('No checks yet.', { exact: true })).toBeVisible();
+  await expect(monitor.getByText('SUCCEEDED', { exact: true })).toHaveCount(0);
+  await expect(monitor.getByText('FAILED', { exact: true })).toHaveCount(0);
+});
+
+test('list failures use the internal error category', async ({ page }) => {
+  await page.route('**/api/monitors', route => route.fulfill({
+    status: 500, json: { error: { code: 'INTERNAL_ERROR', message: 'Arbitrary server prose A' } },
+  }));
+  await page.goto('/monitors');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveAttribute('data-error-code', 'INTERNAL_ERROR');
+  await expect(page.getByRole('main').getByRole('alert')).toHaveText(internalError);
+  await expect(page.getByRole('article')).toHaveCount(0);
+});
+
+test('a malformed error envelope uses a safe category without a successful mutation', async ({ page }) => {
+  await emptyListAndCreateFailure(page, route => route.fulfill({ status: 400, json: { unexpected: true } }));
+  await expectFailedCreate(page, 'INTERNAL_ERROR', internalError);
+});
+
+test('network failure cannot look like a successful mutation', async ({ page }) => {
+  await emptyListAndCreateFailure(page, route => route.abort('failed'));
+  await expectFailedCreate(page, 'INTERNAL_ERROR', internalError);
+});
+
+test('a malformed success payload is rejected before rendering a Monitor', async ({ page }) => {
+  await emptyListAndCreateFailure(page, route => route.fulfill({ status: 201, json: { data: {} } }));
+  await expectFailedCreate(page, 'INTERNAL_ERROR', internalError);
+});


## `fix: reject overflowing numeric monitor intervals`

diff --git a/backend/src/main/java/dev/evolution/monitor/MonitorController.java b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
index 2fa6075..18d963c 100644
--- a/backend/src/main/java/dev/evolution/monitor/MonitorController.java
+++ b/backend/src/main/java/dev/evolution/monitor/MonitorController.java
@@ -72,7 +72,7 @@ public class MonitorController {
             try {
                 // 60.0 has the integer value 60; strings and fractional values are not coerced.
                 interval = body.get("interval").decimalValue().intValueExact();
-            } catch (ArithmeticException error) {
+            } catch (ArithmeticException | NumberFormatException error) {
                 throw invalid("Interval must be an integer from 1 to 86400 seconds.");
             }
             if (interval < 1 || interval > 86400) {
diff --git a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
index 8be16ca..1ee7395 100644
--- a/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
+++ b/backend/src/test/java/dev/evolution/monitor/MonitorFunctionalTest.java
@@ -106,6 +106,18 @@ class MonitorFunctionalTest {
         }
     }
 
+    @Test
+    void rejectsOverflowedNumericIntervalWithoutMutatingMonitors() {
+        JsonNode before = assertDataEnvelope(api.getForEntity("/api/monitors", JsonNode.class), HttpStatus.OK);
+        // Keep the raw number: parsing and reserializing it could change the overflowing token.
+        var response = postJson("{\"name\":\"Fixture monitor\",\"url\":\"http://127.0.0.1:4321/ok\","
+                + "\"interval\":1e309,\"enabled\":true}");
+        assertApiError(response, HttpStatus.BAD_REQUEST, "INVALID_INPUT");
+        assertEquals("Interval must be an integer from 1 to 86400 seconds.",
+                response.getBody().at("/error/message").textValue());
+        assertEquals(before, assertDataEnvelope(api.getForEntity("/api/monitors", JsonNode.class), HttpStatus.OK));
+    }
+
     @Test
     void rejectsInvalidNameLengthAndUrlSyntax() {
         for (String name : new String[]{"", "a".repeat(101), "😀".repeat(51)}) {


## `docs: record E02 numeric overflow repair evidence`

diff --git a/evidence/E02/repair-attempt-2.md b/evidence/E02/repair-attempt-2.md
new file mode 100644
index 0000000..5adf929
--- /dev/null
+++ b/evidence/E02/repair-attempt-2.md
@@ -0,0 +1,48 @@
+# E02 attempt 2: numeric input repair evidence
+
+Branch: `track/industry-spring`. Spec revision: `0a006589477f8ae47bad3faa5510c999cff85ee4`.
+Repair start: `89e1badc7a9a407248167e0d9df1752d0561cfe9`.
+Code/test commit: `e51deb7770ae4790b013259b8112f03b116e3bf6`.
+Executed on 2026-08-28 (Asia/Seoul), with the existing pinned runtimes and dependencies.
+
+## Fixed HTTP probe
+
+The parent froze this exact raw body and the expected 400 / INVALID_INPUT / no Monitor mutation before dispatch. No request parameter or threshold changed. The same GET → POST → GET sequence ran once against the unchanged packaged API, then once against the repaired package. Each API process started with empty memory and was stopped afterward. No fixture request or load was needed.
+
+```sh
+curl --silent --show-error --max-time 10 --request POST \
+  --header 'Content-Type: application/json' \
+  --data-binary '{"name":"Fixture monitor","url":"http://127.0.0.1:4321/ok","interval":1e309,"enabled":true}' \
+  --write-out '\nHTTP_STATUS=%{http_code}\nELAPSED_SECONDS=%{time_total}\n' \
+  http://127.0.0.1:4322/api/monitors
+```
+
+The surrounding GETs used the same URL, 10-second safety timeout, and curl timer. Times below are observations, not acceptance thresholds.
+
+| Phase | Request | HTTP / exact response body | Seconds |
+| --- | --- | --- | ---: |
+| Unchanged baseline | GET before | 200 / `{"data":[]}` | 0.080948 |
+| Unchanged baseline | POST | 500 / `{"error":{"code":"INTERNAL_ERROR","message":"The service could not complete the request."}}` | 0.024088 |
+| Unchanged baseline | GET after | 200 / `{"data":[]}` | 0.001666 |
+| Repaired package | GET before | 200 / `{"data":[]}` | 0.086736 |
+| Repaired package | POST | 400 / `{"error":{"code":"INVALID_INPUT","message":"Interval must be an integer from 1 to 86400 seconds."}}` | 0.026486 |
+| Repaired package | GET after | 200 / `{"data":[]}` | 0.001712 |
+
+Before any production edit, `mvn -B -ntp -f backend/pom.xml -DskipTests package` succeeded once (2.48 seconds, `/usr/bin/time -p`). One local Java source-launch diagnostic against the installed Jackson jars parsed the frozen body into `DoubleNode`; `decimalValue()` threw `NumberFormatException` through `BigDecimal.valueOf` (0.54 seconds). No extra HTTP baseline was executed.
+
+The production change adds only `NumberFormatException` to the existing interval-conversion catch. It does not broaden the global exception handler. The new `MonitorFunctionalTest#rejectsOverflowedNumericIntervalWithoutMutatingMonitors` sends the raw token without a parse/reserialize round trip, verifies the 400 envelope and safe interval message, and compares the complete Monitor list before and after.
+
+## Verification
+
+`/usr/bin/time -p fnm exec --using 24.19.0 npm run verify` ran once and exited 0 in 32.11 seconds. Its four entries were read from `output/verification/runs.jsonl` (starting at `2026-08-27T23:48:58.025Z`):
+
+| Gate | Result | Runner seconds |
+| --- | --- | ---: |
+| `mvn -B -ntp -f backend/pom.xml package` | PASS: 18 tests (4 CheckRunner, 1 MVC boundary, 13 real-HTTP functional), jar built | 7.917 |
+| `npm run typecheck` | PASS | 1.143 |
+| `npm run build` | PASS: production compilation | 7.969 |
+| `npm run test:e2e` | PASS: all 9 Chromium tests, 1 worker, retries 0 | 14.729 |
+
+The added Java regression passed once in 0.012 seconds. Existing checks still cover numeric 60.0, the string "60", interval boundaries, name trimming/UTF-16 length, URL/fixture restrictions, missing IDs, actual HTTP 503, timeout/connection null status, and safe unexpected-error serialization. Both arbitrary-prose browser scenarios and all other original tests were unchanged and passed. Browser tests used the existing development UI harness; the production compilation was a separate gate.
+
+Budget: one unchanged HTTP baseline, one repaired HTTP probe, one full verification; no additional suite invocations, retries, load, benchmarks, sweeps, or profiler runs. Existing `evidence/E02/fixtures.md`, `verification.md`, and `runs.jsonl` remain byte-for-byte unchanged. No frontend, runtime, dependency, infrastructure, or future-Thread changes. All probe servers were stopped.
