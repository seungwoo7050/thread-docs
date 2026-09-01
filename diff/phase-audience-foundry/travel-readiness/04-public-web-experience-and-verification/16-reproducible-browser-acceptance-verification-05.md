## `build(e2e): pin reproducible Playwright cache`

diff --git a/THIRD_PARTY_NOTICES.md b/THIRD_PARTY_NOTICES.md
index 590f97c..336f816 100644
--- a/THIRD_PARTY_NOTICES.md
+++ b/THIRD_PARTY_NOTICES.md
@@ -35,6 +35,37 @@ interpreter를 `-I -S -B scripts/check-dependencies ... --python-bin <같은 abs
 형태로 실행해야 합니다. `scripts/check-dependencies` 자체를 executable로 직접 실행하면
 fail closed합니다.
 
+## browser acceptance용 host toolchain
+
+아래 구성요소는 로컬 browser acceptance를 위한 host-only 도구입니다. application source나
+production artifact에 vendoring하지 않으며, browser runner는 검토된 경로·버전·integrity와
+전체 tree digest가 일치하는 private snapshot만 실행합니다.
+
+| 구성요소 | revision | license 또는 적용 조건 | upstream와 목적 |
+| --- | --- | --- | --- |
+| Node.js | 23.11.0 | MIT 및 배포물에 포함된 제3자 고지 | `https://nodejs.org/`; 고정 Playwright CLI interpreter |
+| `@playwright/cli` | 0.1.18 | Apache-2.0 | `https://www.npmjs.com/package/@playwright/cli`; browser acceptance CLI |
+| `playwright`, `playwright-core` | 1.63.0-alpha-2026-08-05 | Apache-2.0 및 package `NOTICE` | `https://github.com/microsoft/playwright`; browser automation runtime |
+| `fsevents` | 2.3.2 | MIT | optional macOS lock entry; canonical install tree에는 materialize하지 않음 |
+| Google Chrome for Testing | 152.0.7977.8, revision 1237 | Google Chrome 추가 서비스 약관 및 구성요소별 open-source license | `https://googlechromelabs.github.io/chrome-for-testing/`; 테스트 전용 browser binary |
+
+canonical clean npm cache의 `package-lock.json` SHA-256은
+`d12e0bfb9d3fbf1453d06cd48df5b034b246c274574bbf428d780c5328e60abe`, mode 포함 tree
+SHA-256은 `b74893df767491c4c30553fdbe599b23717a2ec1fff3909632245cd97d43d079`, mode 제외
+content SHA-256은 `659591a0079c567eccd1896a280691e63c6bf88068415938f388cb94d57e7767`입니다.
+tree는 242 entries와 19,006,475 regular-file bytes로 고정합니다. lock은 root와 네 개의
+검토된 `node_modules` key만 허용하여 npm alias/history 때문에 생기는 중첩·누적 graph를
+거부합니다. package manifest와 설치된 `LICENSE`/`NOTICE`는 제거하지 않습니다.
+
+Node executable SHA-256은
+`01ca46d5dbf4e6fd39e2cca6154b24e867fae029512b125b1b108ecfbcc1b462`, Chrome executable
+SHA-256은 `72d65943199c16a93085b9d4b11fabb23362c44a2d09fc1d2912565911d3c191`, Chrome bundle
+tree SHA-256은 `35da223dd6f8d25fcd6a7cbc057a84aa7e7ecbdbf9b3911db3d8ce5bdf30ae48`입니다.
+Chrome executable 사용에는 Google Terms of Service와
+`https://www.google.com/chrome/terms/`의 Chrome 추가 약관이 적용되며, open-source
+구성요소 고지는 browser의 `chrome://credits`가 안내합니다. 이 로컬 QA inventory는 향후
+production platform이나 production browser를 선택했다는 의미가 아닙니다.
+
 ## lock에 포함된 Python distribution
 
 | dependency | 관계 | license | upstream | 사용 목적 |
diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index e53ae13..856654c 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -48,11 +48,11 @@ PLAYWRIGHT_WRAPPER: Final = Path(
 WRAPPER_SHA256: Final = "aa3fdff5d0e4556177f4dfd5f04117e772aa54f94b6a2e34b6c0edf629c6b9b5"
 REVIEWED_CLI_VERSION: Final = "0.1.18"
 CACHED_CLI_ROOT: Final = Path("/private/tmp/npm-cache/_npx/31e32ef8478fbf80")
-CACHED_CLI_TREE_SHA256: Final = "7bdee27eb125919be7c20bb115794d933790540a364ef063b17624179db43c0e"
-CACHED_CLI_CONTENT_SHA256: Final = "6a26b64c65840331c735b35a2d562fb7387acb35cfe68484c2c40fb8cce8f368"
-CACHED_CLI_LOCK_SHA256: Final = "39b2c57962abddc563433510a9fbd02c001a5d9957b33dd6615a29839c32881d"
+CACHED_CLI_TREE_SHA256: Final = "b74893df767491c4c30553fdbe599b23717a2ec1fff3909632245cd97d43d079"
+CACHED_CLI_CONTENT_SHA256: Final = "659591a0079c567eccd1896a280691e63c6bf88068415938f388cb94d57e7767"
+CACHED_CLI_LOCK_SHA256: Final = "d12e0bfb9d3fbf1453d06cd48df5b034b246c274574bbf428d780c5328e60abe"
 CACHED_CLI_ENTRY_COUNT: Final = 242
-CACHED_CLI_FILE_BYTES: Final = 20_115_877
+CACHED_CLI_FILE_BYTES: Final = 19_006_475
 PLAYWRIGHT_VERSION: Final = "1.63.0-alpha-2026-08-05"
 PACKAGE_INTEGRITIES: Final = {
     "@playwright/cli": "sha512-ggNfYYH+GsZTGUiBEL8f6N5j0seYEUE52v+fIWqK/A36QG36cL0EJ79qWTXYO2uZMUU7vm+jk3x0fKCPL6UuIw==",
@@ -707,13 +707,16 @@ def _validate_package_lock(root: Path) -> None:
         packages = json.loads(lock.read_text(encoding="utf-8"))["packages"]
     except (OSError, UnicodeError, json.JSONDecodeError, KeyError, TypeError) as exc:
         raise AcceptanceFailure("cli-lock-invalid") from exc
-    found: dict[str, dict[str, object]] = {}
-    for key, value in packages.items():
-        for name in PACKAGE_INTEGRITIES:
-            if key.endswith(f"/node_modules/{name}"):
-                found[name] = value
-    if set(found) != set(PACKAGE_INTEGRITIES):
+    expected_package_keys = {
+        "",
+        *(f"node_modules/{name}" for name in PACKAGE_INTEGRITIES),
+    }
+    if not isinstance(packages, dict) or set(packages) != expected_package_keys:
         raise AcceptanceFailure("cli-lock-package-set")
+    found = {
+        name: packages[f"node_modules/{name}"]
+        for name in PACKAGE_INTEGRITIES
+    }
     versions = {
         "@playwright/cli": REVIEWED_CLI_VERSION,
         "playwright": PLAYWRIGHT_VERSION,
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index ebe0fef..24d9bd7 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -125,6 +125,43 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
             probe.bind(("127.0.0.1", 0))
             return probe.getsockname()[1]
 
+    @staticmethod
+    def reviewed_package_lock() -> dict[str, object]:
+        versions = {
+            "@playwright/cli": acceptance.REVIEWED_CLI_VERSION,
+            "playwright": acceptance.PLAYWRIGHT_VERSION,
+            "playwright-core": acceptance.PLAYWRIGHT_VERSION,
+            "fsevents": "2.3.2",
+        }
+        return {
+            "name": "reviewed-browser-cli",
+            "lockfileVersion": 3,
+            "requires": True,
+            "packages": {
+                "": {"dependencies": {"@playwright/cli": "^0.1.18"}},
+                **{
+                    f"node_modules/{name}": {
+                        "version": versions[name],
+                        "resolved": (
+                            "https://registry.npmjs.org/"
+                            f"{name.replace('@playwright/', '')}/-/reviewed.tgz"
+                        ),
+                        "integrity": integrity,
+                    }
+                    for name, integrity in acceptance.PACKAGE_INTEGRITIES.items()
+                },
+            },
+        }
+
+    @staticmethod
+    def write_reviewed_package_lock(root: Path, payload: dict[str, object]) -> str:
+        lock = root / "package-lock.json"
+        lock.write_text(
+            json.dumps(payload, ensure_ascii=True, indent=2) + "\n",
+            encoding="utf-8",
+        )
+        return hashlib.sha256(lock.read_bytes()).hexdigest()
+
     def make_certificate(self, root: Path) -> tuple[Path, Path, str]:
         certificate = root / "csrf-cert.pem"
         private_key = root / "csrf-private-key.pem"
@@ -161,6 +198,43 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         der = ssl.PEM_cert_to_DER_cert(certificate.read_text(encoding="ascii"))
         return certificate, private_key, acceptance.certificate_spki_sha256(der)
 
+    def test_package_lock_accepts_only_the_reviewed_top_level_graph(self):
+        with tempfile.TemporaryDirectory() as temporary:
+            root = Path(temporary)
+            digest = self.write_reviewed_package_lock(
+                root, self.reviewed_package_lock()
+            )
+            with patch.object(acceptance, "CACHED_CLI_LOCK_SHA256", digest):
+                acceptance._validate_package_lock(root)
+
+    def test_package_lock_rejects_prefixed_and_bloated_graphs(self):
+        for mutation in ("prefixed", "bloated"):
+            with (
+                self.subTest(mutation=mutation),
+                tempfile.TemporaryDirectory() as temporary,
+            ):
+                root = Path(temporary)
+                payload = self.reviewed_package_lock()
+                packages = payload["packages"]
+                self.assertIsInstance(packages, dict)
+                if mutation == "prefixed":
+                    packages["node_modules/@playwright/cli/node_modules/playwright"] = (
+                        packages.pop("node_modules/playwright")
+                    )
+                else:
+                    packages["node_modules/unreviewed"] = {
+                        "version": "1.0.0",
+                        "resolved": "https://registry.npmjs.org/unreviewed/-/unreviewed.tgz",
+                        "integrity": "sha512-unreviewed",
+                    }
+                digest = self.write_reviewed_package_lock(root, payload)
+                with (
+                    patch.object(acceptance, "CACHED_CLI_LOCK_SHA256", digest),
+                    self.assertRaises(acceptance.AcceptanceFailure) as caught,
+                ):
+                    acceptance._validate_package_lock(root)
+                self.assertEqual(caught.exception.check, "cli-lock-package-set")
+
     def wait_for_https(self, port: int, process: subprocess.Popen[bytes]) -> None:
         context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
         context.check_hostname = False


## `fix(e2e): bind browser notices to dependency gate`

diff --git a/scripts/check-dependencies b/scripts/check-dependencies
index 859a2ff..1517f01 100755
--- a/scripts/check-dependencies
+++ b/scripts/check-dependencies
@@ -104,7 +104,7 @@ EXPECTED_PYTHON_BYTECODE = {
     ),
 }
 EXPECTED_NOTICE_SHA256 = (
-    "b10cb31a42abed830c86d8b7f8c399852b5a24a4c3dcf90e955217bbcedf1290"
+    "8f1b56a56ad170b1bcafd53a0897c0c0c0858e0fa84c350e44f62fb4f274a0ec"
 )
 EXPECTED_UV_SEED_FILES = {
     "_virtualenv.py": {


## `test(frontend): renew focused browser acceptance`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index b43b6a6..df655d5 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -2958,7 +2958,7 @@ def _assert_persistent_artifacts(
 ) -> None:
     screenshot_names = (
         FOCUSED_SCREENSHOT_NAMES
-        if receipt.get("evidence_contract") == "travel-opportunities-focused-v1"
+        if receipt.get("evidence_contract") == "travel-opportunities-focused-v2"
         else SCREENSHOT_NAMES
     )
     expected_dirs = {run_root / f"{width}x{height}" for width, height in VIEWPORTS}
@@ -3523,6 +3523,30 @@ def _focused_common_javascript(*, origin: str, check: str) -> str:
   }}), {json.dumps(origin)});
   if (pageState.externalResource) fail('external-resource');
   if (pageState.horizontalOverflow) fail('horizontal-overflow');
+  const typography = await page.evaluate(async () => {{
+    await document.fonts.ready;
+    const body = document.body;
+    const heading = document.querySelector('h1');
+    const loadedResources = performance.getEntriesByType('resource')
+      .map((entry) => new URL(entry.name).pathname)
+      .filter((path) => path.startsWith('/static/public_web/fonts/'))
+      .sort();
+    return {{
+      bodyFamily: getComputedStyle(body).fontFamily,
+      headingFamily: heading ? getComputedStyle(heading).fontFamily : '',
+      maruLoaded: document.fonts.check('600 32px "Maru Buri"', '이번 휴일'),
+      nanumRegularLoaded: document.fonts.check('400 16px "NanumSquare Neo"', '어디 갈까'),
+      nanumBoldLoaded: document.fonts.check('700 16px "NanumSquare Neo"', '갈 수 있는 곳 찾기'),
+      loadedResources,
+    }};
+  }});
+  if (!typography.bodyFamily.includes('NanumSquare Neo') || !typography.headingFamily.includes('Maru Buri')) fail('font-family');
+  if (!typography.maruLoaded || !typography.nanumRegularLoaded || !typography.nanumBoldLoaded) fail('font-load');
+  if (typography.loadedResources.join('|') !== [
+    '/static/public_web/fonts/maru-buri-semibold.woff2',
+    '/static/public_web/fonts/nanum-square-neo-bold.woff2',
+    '/static/public_web/fonts/nanum-square-neo-regular.woff2',
+  ].join('|')) fail('font-resource-set');
   const departure = page.getByLabel('출발 가능한 시각');
   const returning = page.getByLabel('인천 도착 마감 시각');
   if (await departure.count() !== 1 || await returning.count() !== 1) fail('form-label');
@@ -3654,6 +3678,24 @@ def _focused_result_javascript(
       if (await opportunity.locator('.alternatives > ol > li').count() > 2) fail('alternative-count');
       if (!(await opportunity.textContent()).includes('공식 운항 일정') || !(await opportunity.textContent()).includes('예상 시각')) fail('fact-estimate-separation');
     }}
+    const index = page.locator('nav.destination-index[aria-labelledby="destination-index-heading"]');
+    const indexLinks = index.locator('a[href^="#destination-"]');
+    if (await index.count() !== 1 || await indexLinks.count() !== count) fail('destination-index');
+    const indexContract = await page.evaluate(() => {{
+      const links = [...document.querySelectorAll('.destination-index a[href^="#destination-"]')];
+      return links.every((link, position) => {{
+        const target = document.querySelector(link.getAttribute('href'));
+        const indexStay = link.querySelector('.index-stay strong')?.textContent.trim();
+        const detailStay = target?.querySelector('.stay-time strong')?.textContent.trim();
+        const indexCity = link.querySelector('.index-place > strong')?.textContent.trim();
+        const detailCity = target?.querySelector('.destination-heading h3')?.textContent.trim();
+        return target instanceof HTMLElement
+          && target.id === `destination-${{position + 1}}`
+          && indexStay === detailStay
+          && indexCity === detailCity;
+      }});
+    }});
+    if (!indexContract) fail('destination-index-contract');
     const externalLinks = page.locator('a[rel="noopener noreferrer"]');
     if (await externalLinks.count() < 3) fail('source-links');
   }} else if (await page.locator('[data-module="entry"], [data-module="warning"]').count() !== 0) {{
@@ -3761,6 +3803,7 @@ def _focused_accessibility_javascript(*, origin: str, check: str) -> str:
       document.querySelector('label[for="id_return_by"]'),
       document.querySelector('[data-submit-button]'),
       document.querySelector('#results-heading'),
+      document.querySelector('nav[aria-labelledby="destination-index-heading"]'),
       document.querySelector('ol[aria-label="추천 도시"]'),
       document.querySelector('.opportunity h3'),
     ];
@@ -3772,7 +3815,8 @@ def _focused_accessibility_javascript(*, origin: str, check: str) -> str:
       && document.title === '어디 갈까 ?? — 휴일에 갈 수 있는 도시 찾기'
       && nodes[0].tagName === 'H1'
       && nodes[4].tagName === 'H2'
-      && nodes[6].tagName === 'H3'
+      && nodes[5].tagName === 'NAV'
+      && nodes[7].tagName === 'H3'
       && names.join('|') === '후쿠오카|타이베이|다낭'
       && opportunities.every((item) => item.getAttribute('aria-labelledby') === item.querySelector('h3')?.id);
   }});
@@ -3835,7 +3879,7 @@ def run_focused_matrix(
         "checks": [],
         "cli_version": actual_version,
         "completed_at_utc": None,
-        "evidence_contract": "travel-opportunities-focused-v1",
+        "evidence_contract": "travel-opportunities-focused-v2",
         "node": {"version": NODE_VERSION, "sha256": NODE_SHA256},
         "release_sha": release_sha,
         "schema_version": 4,
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index e035a02..e216c58 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -1112,6 +1112,8 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
                 )
         common, validation, correction, result, accessibility = sources
         self.assertIn("performance.getEntriesByType", common)
+        self.assertIn("await document.fonts.ready", common)
+        self.assertIn("font-resource-set", common)
         self.assertIn("await fetch('/',", common)
         self.assertIn("credentials: 'same-origin'", common)
         self.assertIn("headerProbe.status !== 200", common)
@@ -1129,6 +1131,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("entry-state-isolation", result)
         self.assertIn("warning-state-isolation", result)
         self.assertIn("composite-state-boundary", result)
+        self.assertIn("destination-index-contract", result)
         self.assertIn("index === 0 ? 'stale' : 'ready'", result)
         self.assertIn("index === 0 ? 'unavailable' : 'ready'", result)
         self.assertIn(json.dumps(acceptance.FOCUSED_LONG_AIRPORT), accessibility)
@@ -1689,7 +1692,7 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertEqual(receipt["status"], "passed")
         self.assertEqual(receipt["schema_version"], 4)
         self.assertEqual(
-            receipt["evidence_contract"], "travel-opportunities-focused-v1"
+            receipt["evidence_contract"], "travel-opportunities-focused-v2"
         )
         self.assertEqual(len(receipt["screenshots"]), 12)
         self.assertEqual(


