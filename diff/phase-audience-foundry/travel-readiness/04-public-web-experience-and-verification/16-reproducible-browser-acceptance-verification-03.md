## `fix(browser): redact trip inputs from acceptance evidence`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 0e21672..feeb876 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -1774,6 +1774,14 @@ def loading_start_javascript(*, origin: str, check: str) -> str:
     return form?.getAttribute('aria-busy') === 'true' && button?.getAttribute('aria-disabled') === 'true' && button.textContent.includes('제출 중') && status?.textContent.includes('불러오는 중') && status.getAttribute('role') === 'status' && status.getAttribute('aria-live') === 'polite';
   }});
   if (!loading) fail('loading-semantics');
+  const inputsCleared = await page.evaluate(() => {{
+    const departure = document.querySelector('[name="departure_date"]');
+    const returning = document.querySelector('[name="return_date"]');
+    if (!(departure instanceof HTMLInputElement) || !(returning instanceof HTMLInputElement)) return false;
+    departure.value = ''; returning.value = '';
+    return departure.value === '' && returning.value === '';
+  }});
+  if (!inputsCleared) fail('loading-input-redaction');
   {_dom_privacy_source(origin=origin, path='/')}
   {_dynamic_layout_source()}
   return {_marker(check)};
@@ -1938,6 +1946,14 @@ def validation_javascript(*, origin: str, check: str) -> str:
   const field = page.getByLabel('귀국일', {{ exact: true }});
   if (await field.getAttribute('aria-invalid') !== 'true' || !(await field.getAttribute('aria-describedby') || '').includes('id_return_date_error')) fail('validation-description');
   if (page.url() !== {json.dumps(origin + '/')}) fail('validation-location');
+  const inputsCleared = await page.evaluate(() => {{
+    const departure = document.querySelector('[name="departure_date"]');
+    const returning = document.querySelector('[name="return_date"]');
+    if (!(departure instanceof HTMLInputElement) || !(returning instanceof HTMLInputElement)) return false;
+    departure.value = ''; returning.value = '';
+    return departure.value === '' && returning.value === '' && document.activeElement?.matches('[data-error-summary]');
+  }});
+  if (!inputsCleared) fail('validation-input-redaction');
   {_dom_privacy_source(origin=origin, path='/')}
   {_static_asset_integrity_source(origin)}
   {_dynamic_layout_source()}
@@ -1952,6 +1968,7 @@ def correction_javascript(*, origin: str, check: str) -> str:
   if (!await page.evaluate(() => document.activeElement?.matches('[data-error-summary]'))) fail('error-focus-start');
   await page.keyboard.press('Tab'); await page.keyboard.press('Enter');
   if (!await page.evaluate(() => document.activeElement?.id === 'id_return_date')) fail('error-link-target');
+  await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
   await page.keyboard.press('Tab');
   if (!await page.evaluate(() => document.activeElement?.matches('[data-submit-button]'))) fail('correction-order');
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 9b5dfa0..75bebdb 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -516,6 +516,21 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("finishSubmit(200, false)", acceptance.validation_javascript(
             origin=self.origins["ready"], check="validation"
         ))
+        loading_source = acceptance.loading_start_javascript(
+            origin=self.origins["loading"], check="loading"
+        )
+        validation_source = acceptance.validation_javascript(
+            origin=self.origins["ready"], check="validation"
+        )
+        correction_source = acceptance.correction_javascript(
+            origin=self.origins["ready"], check="correction"
+        )
+        self.assertIn("loading-input-redaction", loading_source)
+        self.assertIn("validation-input-redaction", validation_source)
+        self.assertIn("departure.value = ''; returning.value = '';", loading_source)
+        self.assertIn("departure.value = ''; returning.value = '';", validation_source)
+        self.assertIn(acceptance.SYNTHETIC_DEPARTURE, correction_source)
+        self.assertIn(acceptance.SYNTHETIC_VALID_RETURN, correction_source)
         self.assertIn("await route.continue();", source)
         self.assertNotIn("route.continue({ headers", source)
         self.assertNotIn("const csrf =", source)


## `fix(e2e): isolate scenario browser state`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 1fd6b6f..9b4547c 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -1867,12 +1867,18 @@ def form_pristine_javascript(*, origin: str, check: str) -> str:
 
 
 def _submission_support(origin: str) -> str:
+    expected_host = urlsplit(
+        validate_base_url(origin, field="submission-support")
+    ).hostname
+    if expected_host is None:
+        raise AcceptanceFailure("invalid-submission-support")
     return f"""{_client_state_source()}
   const validateSubmitCookie = async () => {{
     await assertClientState();
     const context = page.context();
     const cookies = await context.cookies();
-    if (cookies.length !== 1 || cookies[0].name !== 'csrftoken' || !cookies[0].value || !cookies[0].secure || !cookies[0].httpOnly || cookies[0].sameSite !== 'Strict') fail('csrf-cookie-contract');
+    const expectedHost = {json.dumps(expected_host)};
+    if (cookies.length !== 1 || cookies[0].name !== 'csrftoken' || !cookies[0].value || !cookies[0].secure || !cookies[0].httpOnly || cookies[0].sameSite !== 'Strict' || cookies[0].domain !== expectedHost || cookies[0].path !== '/') fail('csrf-cookie-contract');
   }};
   const installSubmitObservers = async () => {{
     const guard = page.context().__acceptanceGuard;
@@ -2044,6 +2050,46 @@ def csrf_contract_probe_javascript(
 }}"""
 
 
+def csrf_cookie_contract_javascript(*, origin: str, check: str) -> str:
+    """Verify one browser-owned CSRF cookie without exposing its value."""
+
+    validate_base_url(origin, field="csrf-cookie-contract")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_submission_support(origin)}
+  if (page.url() !== {json.dumps(origin + '/')}) fail('csrf-form-location');
+  await validateSubmitCookie();
+  return {_marker(check)};
+}}"""
+
+
+def isolate_origin_javascript(*, origin: str, check: str) -> str:
+    """Clear browser state before crossing to another scenario origin."""
+
+    expected_host = urlsplit(
+        validate_base_url(origin, field="origin-isolation")
+    ).hostname
+    if expected_host is None:
+        raise AcceptanceFailure("invalid-origin-isolation")
+    return f"""async (page) => {{
+  const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
+  {_client_state_source()}
+  if (![{json.dumps(origin + '/')}, {json.dumps(origin + '/results/')}].includes(page.url())) fail('isolation-origin');
+  if (page.__acceptanceSubmit) fail('isolation-submit-state');
+  const guard = page.context().__acceptanceGuard;
+  if (!guard || guard.external !== 0 || guard.unexpected !== 0) fail('request-attempt');
+  await assertClientState();
+  const cookies = await page.context().cookies();
+  const expectedHost = {json.dumps(expected_host)};
+  if (cookies.length > 1 || cookies.some((cookie) => cookie.name !== 'csrftoken' || !cookie.value || !cookie.secure || !cookie.httpOnly || cookie.sameSite !== 'Strict' || cookie.domain !== expectedHost || cookie.path !== '/')) fail('isolation-cookie-state');
+  await page.evaluate(() => {{ localStorage.clear(); sessionStorage.clear(); }});
+  await page.context().clearCookies();
+  if ((await page.context().cookies()).length !== 0) fail('cross-origin-cookie-carryover');
+  await assertClientState();
+  return {_marker(check)};
+}}"""
+
+
 def csrf_keyboard_observe_javascript(*, origin: str, check: str) -> str:
     """Install exact POST/response accounting after cookie validation."""
 
@@ -2130,7 +2176,11 @@ def validation_press_javascript(*, origin: str, check: str) -> str:
   const responseReady = page.waitForResponse((response) => response.request().method() === 'POST' && response.url() === {json.dumps(origin + '/')});
   await page.keyboard.press('Enter');
   const response = await responseReady;
-  if (response.status() !== 200) fail('validation-response-status');
+  const status = response.status();
+  if (status === 303) fail('validation-response-redirect');
+  if (status === 403) fail('validation-response-csrf');
+  if (status === 503) fail('validation-response-unavailable');
+  if (status !== 200) fail('validation-response-other');
   await page.waitForLoadState('networkidle');
   if (page.url() !== {json.dumps(origin + '/')}) fail('validation-location');
   return {_marker(check)};
@@ -2817,6 +2867,12 @@ def run_matrix(
                 "form-pristine",
                 form_pristine_javascript(origin=ready, check="form-pristine"),
             )
+            session.run_code(
+                "form-pristine-csrf-cookie",
+                csrf_cookie_contract_javascript(
+                    origin=ready, check="form-pristine-csrf-cookie"
+                ),
+            )
             checks.append({"check": "form-semantics-keyboard", "viewport": viewport})
             screenshots.append(
                 _screenshot(
@@ -2824,6 +2880,12 @@ def run_matrix(
                     scenario="form-pristine", width=width, height=height,
                 )
             )
+            session.run_code(
+                "isolate-ready-to-loading",
+                isolate_origin_javascript(
+                    origin=ready, check="isolate-ready-to-loading"
+                ),
+            )
 
             loading = origins["loading"]
             session.goto(f"{loading}/", check="goto-loading")
@@ -2833,6 +2895,12 @@ def run_matrix(
                     origin=loading, release_sha=release_sha, check="release-loading-form"
                 ),
             )
+            session.run_code(
+                "form-loading-csrf-cookie",
+                csrf_cookie_contract_javascript(
+                    origin=loading, check="form-loading-csrf-cookie"
+                ),
+            )
             loading_relative = Path(viewport) / "form-loading.png"
             loading_target = run_root / loading_relative
             session.validate_screenshot_target(
@@ -2855,6 +2923,12 @@ def run_matrix(
             checks.append(
                 {"check": "loading-trusted-keyboard-busy-post-free", "viewport": viewport}
             )
+            session.run_code(
+                "isolate-loading-to-ready",
+                isolate_origin_javascript(
+                    origin=loading, check="isolate-loading-to-ready"
+                ),
+            )
 
             session.goto(f"{ready}/", check="goto-validation")
             session.run_code(
@@ -2942,7 +3016,16 @@ def run_matrix(
             )
             session.run_code("reset-media", reset_media_javascript(check="reset-media"))
 
+            prior_state = "ready"
             for state in ("empty", "unavailable", "stale", "server-error", "long-korean"):
+                prior_origin = origins[prior_state]
+                session.run_code(
+                    f"isolate-{prior_state}-to-{state}",
+                    isolate_origin_javascript(
+                        origin=prior_origin,
+                        check=f"isolate-{prior_state}-to-{state}",
+                    ),
+                )
                 origin = origins[state]
                 session.goto(f"{origin}/results/", check=f"goto-{state}")
                 session.run_code(
@@ -2965,6 +3048,7 @@ def run_matrix(
                         scenario=state, width=width, height=height,
                     )
                 )
+                prior_state = state
 
             long_origin = origins["long-korean"]
             session.run_code(
@@ -2980,6 +3064,15 @@ def run_matrix(
                 )
             )
             checks.append({"check": "long-korean-200-percent", "viewport": viewport})
+            session.run_code(
+                "isolate-long-korean-to-ready",
+                isolate_origin_javascript(
+                    origin=long_origin, check="isolate-long-korean-to-ready"
+                ),
+            )
+            checks.append(
+                {"check": "cross-origin-cookie-storage-isolation", "viewport": viewport}
+            )
             session.goto(f"{ready}/", check="goto-scaled-form")
             session.run_code(
                 "release-scaled-form",
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 0875e4c..3d55e67 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -480,6 +480,12 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
             acceptance.csrf_keyboard_finish_javascript(
                 origin=self.origins["ready"], check="csrf-finish"
             ),
+            acceptance.csrf_cookie_contract_javascript(
+                origin=self.origins["ready"], check="csrf-cookie"
+            ),
+            acceptance.isolate_origin_javascript(
+                origin=self.origins["ready"], check="isolate-origin"
+            ),
             *(
                 acceptance.csrf_contract_probe_javascript(
                     origin=self.origins["ready"], aspect=aspect,
@@ -568,7 +574,10 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         )
         self.assertIn("finishSubmit(200, false)", validation_source)
         self.assertIn("response.request().method() === 'POST'", validation_source)
-        self.assertIn("response.status() !== 200", validation_source)
+        self.assertIn("status === 403", validation_source)
+        self.assertIn("status === 303", validation_source)
+        self.assertIn("status === 503", validation_source)
+        self.assertIn("status !== 200", validation_source)
         self.assertIn("page.waitForLoadState('networkidle')", validation_source)
         correction_source = submit_source
         self.assertIn("loading-input-redaction", loading_source)
@@ -602,6 +611,25 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("dom-privacy-", form_source)
         self.assertIn("crypto.subtle.digest('SHA-256'", form_source)
 
+        cookie_source = acceptance.csrf_cookie_contract_javascript(
+            origin=self.origins["ready"], check="csrf-cookie"
+        )
+        isolation_source = acceptance.isolate_origin_javascript(
+            origin=self.origins["ready"], check="isolate-origin"
+        )
+        self.assertIn("cookies.length !== 1", cookie_source)
+        self.assertIn("cookies[0].secure", cookie_source)
+        self.assertIn("cookies[0].httpOnly", cookie_source)
+        self.assertIn("cookies[0].sameSite !== 'Strict'", cookie_source)
+        self.assertIn("cookies[0].domain !== expectedHost", cookie_source)
+        self.assertIn("cookies[0].path !== '/'", cookie_source)
+        self.assertIn("localStorage.clear(); sessionStorage.clear();", isolation_source)
+        self.assertIn("page.context().clearCookies()", isolation_source)
+        self.assertIn("cross-origin-cookie-carryover", isolation_source)
+        self.assertIn("await assertClientState();", isolation_source)
+        self.assertNotIn("postData", isolation_source)
+        self.assertNotIn("request.body", isolation_source)
+
     def test_pristine_tab_order_only_allows_bounded_native_date_segments(self):
         source = acceptance.form_pristine_javascript(
             origin=self.origins["ready"], check="tab-order"
@@ -1019,6 +1047,38 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
             ),
             0,
         )
+        self.assertEqual(
+            sum(
+                check == "form-pristine-csrf-cookie" and command == "run-code"
+                for check, command in calls
+            ),
+            4,
+        )
+        self.assertEqual(
+            sum(
+                check == "form-loading-csrf-cookie" and command == "run-code"
+                for check, command in calls
+            ),
+            4,
+        )
+        transitions = (
+            "isolate-ready-to-loading",
+            "isolate-loading-to-ready",
+            "isolate-ready-to-empty",
+            "isolate-empty-to-unavailable",
+            "isolate-unavailable-to-stale",
+            "isolate-stale-to-server-error",
+            "isolate-server-error-to-long-korean",
+            "isolate-long-korean-to-ready",
+        )
+        for transition in transitions:
+            self.assertEqual(
+                sum(
+                    check == transition and command == "run-code"
+                    for check, command in calls
+                ),
+                4,
+            )
         receipt_checks = {
             (item["check"], item["viewport"]) for item in receipt["checks"]
         }
@@ -1034,6 +1094,10 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
                 ),
                 receipt_checks,
             )
+            self.assertIn(
+                ("cross-origin-cookie-storage-isolation", viewport),
+                receipt_checks,
+            )
 
     def test_toolchain_wrapper_and_real_https_django_csrf_submit_leave_no_residue(self):
         """Real Chrome and Django; run with local GUI/process permission."""
@@ -1130,30 +1194,38 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
 
             certificate, private_key, certificate_spki = self.make_certificate(root)
             result_file = root / "csrf-result.txt"
-            port = self.unused_port()
-            origin = f"https://127.0.0.1:{port}"
-            csrf_process = subprocess.Popen(
-                [
-                    str(acceptance.REPOSITORY_ROOT / ".venv" / "bin" / "python"),
-                    "-I", "-B", "-c", DJANGO_CSRF_SERVER_CODE,
-                    str(port), str(certificate), str(private_key), str(result_file),
-                ],
-                cwd=root,
-                env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C", "TZ": "UTC"},
-                stdin=subprocess.DEVNULL,
-                stdout=subprocess.DEVNULL,
-                stderr=subprocess.DEVNULL,
-                start_new_session=True,
-            )
+            ports = (self.unused_port(), self.unused_port())
+            while ports[1] == ports[0]:
+                ports = (ports[0], self.unused_port())
+            first_origin = f"https://127.0.0.1:{ports[0]}"
+            origin = f"https://127.0.0.1:{ports[1]}"
+            csrf_processes = [
+                subprocess.Popen(
+                    [
+                        str(acceptance.REPOSITORY_ROOT / ".venv" / "bin" / "python"),
+                        "-I", "-B", "-c", DJANGO_CSRF_SERVER_CODE,
+                        str(port), str(certificate), str(private_key), str(result_file),
+                    ],
+                    cwd=root,
+                    env={"PATH": "/usr/bin:/bin", "LANG": "C", "LC_ALL": "C", "TZ": "UTC"},
+                    stdin=subprocess.DEVNULL,
+                    stdout=subprocess.DEVNULL,
+                    stderr=subprocess.DEVNULL,
+                    start_new_session=True,
+                )
+                for port in ports
+            ]
             try:
-                self.wait_for_https(port, csrf_process)
+                for port, process in zip(ports, csrf_processes, strict=True):
+                    self.wait_for_https(port, process)
+                acceptance.verify_tls_peer(first_origin, certificate_spki)
                 acceptance.verify_tls_peer(origin, certificate_spki)
                 other_ports: list[int] = []
-                while len(other_ports) < 6:
+                while len(other_ports) < 5:
                     candidate = self.unused_port()
-                    if candidate != port and candidate not in other_ports:
+                    if candidate not in ports and candidate not in other_ports:
                         other_ports.append(candidate)
-                csrf_origins = [origin] + [
+                csrf_origins = [first_origin, origin] + [
                     f"https://127.0.0.1:{candidate}" for candidate in other_ports
                 ]
                 csrf_config = root / "csrf-config.json"
@@ -1182,7 +1254,26 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
                             origins=csrf_origins, check="csrf-install-guards"
                         ),
                     )
+                    csrf_session.goto(first_origin + "/", check="csrf-goto-first-form")
+                    csrf_session.run_code(
+                        "csrf-first-cookie",
+                        acceptance.csrf_cookie_contract_javascript(
+                            origin=first_origin, check="csrf-first-cookie"
+                        ),
+                    )
+                    csrf_session.run_code(
+                        "csrf-cross-origin-isolation",
+                        acceptance.isolate_origin_javascript(
+                            origin=first_origin, check="csrf-cross-origin-isolation"
+                        ),
+                    )
                     csrf_session.goto(origin + "/", check="csrf-goto-form")
+                    csrf_session.run_code(
+                        "csrf-second-cookie",
+                        acceptance.csrf_cookie_contract_javascript(
+                            origin=origin, check="csrf-second-cookie"
+                        ),
+                    )
                     csrf_session.run_code(
                         "csrf-location-exact",
                         "async (page) => { "
@@ -1230,7 +1321,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
                             raise acceptance.AcceptanceFailure(
                                 "csrf-regression-post-received"
                             ) from exc
-                        if csrf_process.poll() is not None:
+                        if any(process.poll() is not None for process in csrf_processes):
                             raise acceptance.AcceptanceFailure(
                                 "csrf-regression-server-exited"
                             ) from exc
@@ -1253,12 +1344,14 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
                     csrf_session.close()
                 self.assertEqual(result_file.read_text(encoding="ascii"), "1\n")
             finally:
-                csrf_process.terminate()
-                try:
-                    csrf_process.wait(timeout=5)
-                except subprocess.TimeoutExpired:
-                    csrf_process.kill()
-                    csrf_process.wait(timeout=5)
+                for process in csrf_processes:
+                    process.terminate()
+                for process in csrf_processes:
+                    try:
+                        process.wait(timeout=5)
+                    except subprocess.TimeoutExpired:
+                        process.kill()
+                        process.wait(timeout=5)
             self.assertEqual(
                 acceptance._verify_sealed_toolchain_tree(toolchain.sealed_root),
                 (


