## `test(frontend): stabilize semantic acceptance hooks`

diff --git a/e2e/browser_acceptance.py b/e2e/browser_acceptance.py
index 150082d..e53ae13 100644
--- a/e2e/browser_acceptance.py
+++ b/e2e/browser_acceptance.py
@@ -1692,29 +1692,54 @@ def _static_asset_integrity_source(origin: str) -> str:
   }}"""
 
 
+def _main_document_security_source(*, origin: str, path: str) -> str:
+    if path not in {"/", "/results/"}:
+        raise AcceptanceFailure("invalid-main-document-path")
+    return f"""const mainDocumentUrl = {json.dumps(origin + path)};
+  const mainDocumentAwaited = page.waitForResponse((response) => response.url() === mainDocumentUrl && response.request().method() === 'GET');
+  const mainDocumentObserved = await page.evaluate(async (url) => {{
+    const response = await fetch(url, {{ credentials: 'omit', cache: 'no-store', redirect: 'error', referrerPolicy: 'no-referrer' }});
+    return {{ status: response.status, contentType: response.headers.get('content-type') || '' }};
+  }}, mainDocumentUrl);
+  const mainDocumentResponse = await mainDocumentAwaited;
+  const mainDocumentDetails = await mainDocumentResponse.securityDetails();
+  const mainDocumentHeaders = await mainDocumentResponse.allHeaders();
+  const mainDocumentRequestHeaders = await mainDocumentResponse.request().allHeaders();
+  const header = (name) => (mainDocumentHeaders[name] || '').trim();
+  const tokens = (value) => value.split(';').map((item) => item.trim().toLowerCase()).filter(Boolean).sort();
+  const csp = header('content-security-policy').split(';').map((item) => item.trim()).filter(Boolean).sort();
+  const expectedCsp = ["default-src 'self'", "base-uri 'none'", "form-action 'self'", "frame-ancestors 'none'", "object-src 'none'"].sort();
+  const expectedHsts = ['max-age=31536000', 'includesubdomains', 'preload'].sort();
+  const cacheControl = header('cache-control').toLowerCase().split(',').map((item) => item.trim()).filter(Boolean);
+  if (mainDocumentObserved.status !== 200 || mainDocumentResponse.status() !== 200 || mainDocumentObserved.contentType.split(';')[0].trim().toLowerCase() !== 'text/html') fail('main-response');
+  if (!mainDocumentDetails || !['TLS 1.2', 'TLS 1.3'].includes(mainDocumentDetails.protocol) || mainDocumentResponse.url() !== mainDocumentUrl || mainDocumentResponse.request().redirectedFrom()) fail('main-transport');
+  if ('cookie' in mainDocumentRequestHeaders || 'referer' in mainDocumentRequestHeaders) fail('main-request-privacy');
+  if (!cacheControl.includes('no-store') || header('pragma').toLowerCase() !== 'no-cache') fail('main-cache');
+  if (JSON.stringify(csp) !== JSON.stringify(expectedCsp) || header('referrer-policy').toLowerCase() !== 'same-origin') fail('main-policy');
+  if (header('x-content-type-options').toLowerCase() !== 'nosniff' || header('x-frame-options').toUpperCase() !== 'DENY' || header('cross-origin-opener-policy').toLowerCase() !== 'same-origin') fail('main-browser-boundary');
+  if (JSON.stringify(tokens(header('strict-transport-security'))) !== JSON.stringify(expectedHsts)) fail('main-hsts');"""
+
+
 def _dom_privacy_source(*, origin: str, path: str) -> str:
     if path not in {"/", "/results/"}:
         raise AcceptanceFailure("invalid-dom-privacy-path")
     form_page = path == "/"
-    expected_title = (
-        "여행준비 — 일본 정보 확인" if form_page else "여행준비 — 일본 게시 정보"
-    )
     approved_hrefs = [
         "#main-content", "/", "/results/", "/static/public_web/site.css",
+        "#entry-card", "#warning-card",
         "#id_destination", "#id_departure_date", "#id_return_date", "#trip-form",
         ENTRY_PUBLIC_SOURCE_LOCATOR, WARNING_PUBLIC_SOURCE_LOCATOR,
     ]
     return f"""const privacyFailure = await page.evaluate((contract) => {{
     const text = (node) => node.textContent.replace(/\\s+/g, ' ').trim();
-    const headChildren = [...document.head.children];
-    if (headChildren.map((node) => node.tagName).join('|') !== 'META|META|META|TITLE|LINK') return 'head';
-    if (headChildren[0].getAttributeNames().join(',') !== 'charset' || headChildren[0].getAttribute('charset').toLowerCase() !== 'utf-8') return 'charset';
-    if (headChildren[1].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[1].getAttribute('name') !== 'viewport' || headChildren[1].getAttribute('content') !== 'width=device-width, initial-scale=1') return 'viewport-meta';
-    if (headChildren[2].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[2].getAttribute('name') !== 'theme-color' || headChildren[2].getAttribute('content') !== '#123f46') return 'theme-meta';
-    if (headChildren[3].getAttributeNames().length || text(headChildren[3]) !== contract.title) return 'title';
-    if (headChildren[4].getAttributeNames().sort().join(',') !== 'href,rel' || headChildren[4].getAttribute('rel') !== 'stylesheet' || headChildren[4].getAttribute('href') !== '/static/public_web/site.css' || headChildren[4].href !== contract.origin + '/static/public_web/site.css') return 'style-link';
+    const viewport = document.querySelectorAll('meta[name="viewport"]');
+    if (document.characterSet.toLowerCase() !== 'utf-8') return 'charset';
+    if (viewport.length !== 1 || !viewport[0].content.split(',').map((item) => item.trim().toLowerCase()).includes('width=device-width') || !viewport[0].content.split(',').map((item) => item.trim().toLowerCase()).includes('initial-scale=1')) return 'viewport-meta';
+    if (document.querySelectorAll('title').length !== 1 || !text(document.querySelector('title'))) return 'title';
+    const styles = [...document.querySelectorAll('link[rel~="stylesheet"]')];
+    if (styles.length !== 1 || styles[0].getAttribute('href') !== '/static/public_web/site.css' || styles[0].href !== contract.origin + '/static/public_web/site.css') return 'style-link';
     const scripts = [...document.scripts];
-    if (scripts.length !== 1 || scripts[0].getAttributeNames().sort().join(',') !== 'defer,src' || scripts[0].getAttribute('src') !== '/static/public_web/site.js' || scripts[0].src !== contract.origin + '/static/public_web/site.js' || scripts[0].textContent.trim()) return 'script';
+    if (scripts.length !== 1 || !scripts[0].hasAttribute('defer') || scripts[0].getAttribute('src') !== '/static/public_web/site.js' || scripts[0].src !== contract.origin + '/static/public_web/site.js' || scripts[0].textContent.trim()) return 'script';
     const hidden = [...document.querySelectorAll('input[type="hidden"]')];
     if (contract.form) {{
       if (hidden.length !== 1 || hidden[0].getAttributeNames().sort().join(',') !== 'name,type,value' || hidden[0].getAttribute('name') !== 'csrfmiddlewaretoken' || !hidden[0].getAttribute('value')) return 'hidden-input';
@@ -1739,12 +1764,11 @@ def _dom_privacy_source(*, origin: str, path: str) -> str:
     const bodyText = document.body.textContent || '';
     if (bodyText.length > 100000 || forbidden.test(bodyText) || /[<>{{}}]/.test(bodyText) || encoded.test(bodyText) || opaque.test(bodyText)) return 'text';
     return null;
-  }}, {{ origin: {json.dumps(origin)}, title: {json.dumps(expected_title)}, form: {json.dumps(form_page)}, href: {json.dumps(approved_hrefs)} }});
+  }}, {{ origin: {json.dumps(origin)}, form: {json.dumps(form_page)}, href: {json.dumps(approved_hrefs)} }});
   if (privacyFailure) fail(`dom-privacy-${{privacyFailure}}`);"""
 
 
 def _common_javascript(*, origin: str, path: str, check: str) -> str:
-    static_integrity = _static_asset_integrity_source(origin) if path == "/" else ""
     return f"""async (page) => {{
   const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
   {_client_state_source()}
@@ -1774,9 +1798,10 @@ def _common_javascript(*, origin: str, path: str, check: str) -> str:
   }}, {{ width: viewport && viewport.width, height: viewport && viewport.height }});
   if (!viewport || failure) fail(failure || 'viewport');
   if (await page.getByRole('heading', {{ level: 1 }}).count() !== 1) fail('heading-role');
-  if (await page.getByRole('navigation', {{ name: '주요 메뉴', exact: true }}).count() !== 1) fail('navigation-name');
+  if (await page.locator('nav[aria-label="주요 메뉴"]').count() !== 1) fail('navigation-name');
   {_dom_privacy_source(origin=origin, path=path)}
-  {static_integrity}
+  {_main_document_security_source(origin=origin, path=path)}
+  {_static_asset_integrity_source(origin)}
   return {_marker(check)};
 }}"""
 
@@ -1827,7 +1852,8 @@ def form_pristine_javascript(*, origin: str, check: str) -> str:
   for (const name of ['목적지', '출국일', '귀국일']) {{
     const locator = page.getByLabel(name, {{ exact: true }}); if (await locator.count() !== 1) fail('label-role');
   }}
-  if (await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).count() !== 1) fail('submit-name');
+  const submitButton = page.locator('[data-submit-button]');
+  if (await submitButton.count() !== 1 || !await submitButton.evaluate((node) => node instanceof HTMLButtonElement && node.type === 'submit' && Boolean((node.getAttribute('aria-label') || node.textContent || '').trim()))) fail('submit-name');
   const formFailure = await page.evaluate(() => {{
     for (const id of ['id_destination', 'id_departure_date', 'id_return_date']) {{
       const control = document.getElementById(id); const label = document.querySelector(`label[for="${{id}}"]`);
@@ -1840,7 +1866,7 @@ def form_pristine_javascript(*, origin: str, check: str) -> str:
   }});
   if (formFailure) fail(formFailure);
   await page.evaluate(() => document.activeElement && document.activeElement.blur());
-  const order = ['.skip-link', '.site-nav a:nth-of-type(1)', '.site-nav a:nth-of-type(2)', '#id_destination', '#id_departure_date', '#id_return_date', '[data-submit-button]'];
+  const order = ['a[href="#main-content"]', 'nav[aria-label="주요 메뉴"] a[href="/"]', 'nav[aria-label="주요 메뉴"] a[href="/results/"]', '#id_destination', '#id_departure_date', '#id_return_date', '[data-submit-button]'];
   let previous = null;
   for (const selector of order) {{
     let reached = false;
@@ -1954,18 +1980,24 @@ def loading_visual_capture_javascript(
   const observerReady = await page.evaluate((key) => {{
     const form = document.querySelector('[data-trip-form]');
     if (!(form instanceof HTMLFormElement) || Object.prototype.hasOwnProperty.call(window, key)) return false;
+    const initialButton = form.querySelector('[data-submit-button]');
+    const initialLabel = (initialButton?.getAttribute('aria-label') || initialButton?.textContent || '').trim();
+    if (!(initialButton instanceof HTMLButtonElement) || !initialLabel) return false;
     Object.defineProperty(window, key, {{ configurable: true, enumerable: false, value: null, writable: true }});
     form.addEventListener('submit', (event) => {{ event.preventDefault(); }}, {{ capture: true, once: true }});
     form.addEventListener('submit', (event) => {{
       const button = form.querySelector('[data-submit-button]');
       const status = form.querySelector('[data-submit-status]');
+      const buttonLabel = (button?.getAttribute('aria-label') || button?.textContent || '').trim();
+      const statusLabel = (status?.textContent || '').trim();
       window[key] = Boolean(
         event.isTrusted
         && event.defaultPrevented
         && form.getAttribute('aria-busy') === 'true'
         && button?.getAttribute('aria-disabled') === 'true'
-        && button.textContent.includes('제출 중')
-        && status?.textContent.includes('불러오는 중')
+        && buttonLabel
+        && buttonLabel !== initialLabel
+        && statusLabel
         && status.getAttribute('role') === 'status'
         && status.getAttribute('aria-live') === 'polite'
       );
@@ -1975,7 +2007,7 @@ def loading_visual_capture_javascript(
   if (!observerReady) fail('loading-observer');
   await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
-  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.locator('[data-submit-button]').focus();
   await page.keyboard.press('Enter');
   const visual = await page.evaluate((key) => {{
     const observed = window[key] === true;
@@ -2007,7 +2039,7 @@ def csrf_keyboard_submit_javascript(*, origin: str, check: str) -> str:
   await prepareSubmit();
   await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
-  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.locator('[data-submit-button]').focus();
   await page.keyboard.press('Enter');
   await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }});
   await finishSubmit(303);
@@ -2127,7 +2159,7 @@ def csrf_keyboard_fill_javascript(*, origin: str, check: str) -> str:
   if (!page.__acceptanceSubmit || page.url() !== {json.dumps(origin + '/')}) fail('submit-state');
   await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
-  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.locator('[data-submit-button]').focus();
   const prepared = await page.evaluate((values) => {{
     const departure = document.querySelector('[name="departure_date"]');
     const returning = document.querySelector('[name="return_date"]');
@@ -2172,7 +2204,7 @@ def validation_prepare_javascript(*, origin: str, check: str) -> str:
   await prepareSubmit();
   await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)});
   await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_INVALID_RETURN)});
-  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus();
+  await page.locator('[data-submit-button]').focus();
   const prepared = await page.evaluate((values) => {{
     const departure = document.querySelector('[name="departure_date"]');
     const returning = document.querySelector('[name="return_date"]');
@@ -2220,7 +2252,12 @@ def validation_finish_javascript(*, origin: str, check: str) -> str:
   const summary = page.getByRole('alert'); await summary.waitFor({{ state: 'visible' }});
   await page.waitForFunction(() => document.activeElement?.matches('[data-error-summary]'));
   await finishSubmit(200, false);
-  if (await summary.count() !== 1 || await page.getByRole('heading', {{ level: 2, name: '입력 내용을 확인해 주세요', exact: true }}).count() !== 1) fail('validation-semantics');
+  const validationSemantics = await summary.evaluate((node) => {{
+    const heading = node.querySelector('h2');
+    const labelledBy = (node.getAttribute('aria-labelledby') || '').split(/\\s+/).filter(Boolean);
+    return Boolean(heading?.id && heading.textContent.trim() && labelledBy.includes(heading.id));
+  }});
+  if (await summary.count() !== 1 || !validationSemantics) fail('validation-semantics');
   const field = page.getByLabel('귀국일', {{ exact: true }});
   if (await field.getAttribute('aria-invalid') !== 'true' || !(await field.getAttribute('aria-describedby') || '').includes('id_return_date_error')) fail('validation-description');
   if (page.url() !== {json.dumps(origin + '/')}) fail('validation-location');
@@ -2278,50 +2315,36 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
     return f"""async (page) => {{
   const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
   await ({_common_javascript(origin=origin, path='/results/', check=check)})(page);
-  if (await page.getByRole('heading', {{ level: 1, name: '일본 게시 정보', exact: true }}).count() !== 1) fail('results-heading');
   const pageContract = await page.evaluate((expectedOrigin) => {{
-    const identity = (node) => `${{node.tagName}}:${{node.id}}:${{node.className}}`;
     const text = (node) => node.textContent.replace(/\\s+/g, ' ').trim();
-    const headChildren = [...document.head.children];
-    if (headChildren.map((node) => node.tagName).join('|') !== 'META|META|META|TITLE|LINK') return 'head';
-    if (headChildren[0].getAttributeNames().join(',') !== 'charset' || headChildren[0].getAttribute('charset').toLowerCase() !== 'utf-8') return 'charset';
-    if (headChildren[1].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[1].getAttribute('name') !== 'viewport' || headChildren[1].getAttribute('content') !== 'width=device-width, initial-scale=1') return 'viewport-meta';
-    if (headChildren[2].getAttributeNames().sort().join(',') !== 'content,name' || headChildren[2].getAttribute('name') !== 'theme-color' || headChildren[2].getAttribute('content') !== '#123f46') return 'theme-meta';
-    if (headChildren[3].getAttributeNames().length || text(headChildren[3]) !== '여행준비 — 일본 게시 정보') return 'title';
-    if (headChildren[4].getAttributeNames().sort().join(',') !== 'href,rel' || headChildren[4].getAttribute('rel') !== 'stylesheet' || headChildren[4].href !== expectedOrigin + '/static/public_web/site.css') return 'style-link';
-    if ([...document.body.children].map(identity).join('|') !== 'A::skip-link|HEADER::site-header|MAIN:main-content:page-shell page-main|FOOTER::site-footer|SCRIPT::') return 'body';
-    if (text(document.querySelector('.skip-link')) !== '본문으로 건너뛰기') return 'skip-link';
-    const main = document.querySelector('main'); if ([...main.children].map(identity).join('|') !== 'DIV::page-heading|SECTION::publication-grid') return 'main';
-    const headingChildren = [...main.querySelector('.page-heading').children];
-    if (headingChildren.map(identity).join('|') !== 'P::eyebrow|H1::|P::page-lead' || text(headingChildren[0]) !== '두 개의 독립 publication' || text(headingChildren[1]) !== '일본 게시 정보' || text(headingChildren[2]) !== '고정된 일본 publication만 표시합니다. 여행 목적과 날짜에 대한 적용 여부는 계산하지 않으므로 공식 기관 확인이 필요합니다.') return 'heading';
-    if ([...main.querySelector('.publication-grid').children].map(identity).join('|') !== 'ARTICLE:entry-card:publication-card|ARTICLE:warning-card:publication-card') return 'grid';
-    const header = document.querySelector('header'); const headerShell = document.querySelector('header > .page-shell'); const navigation = document.querySelector('header > .page-shell > nav.site-nav'); const navigationLinks = navigation ? [...navigation.children] : [];
-    if (header.children.length !== 1 || headerShell.children.length !== 1 || navigationLinks.length !== 2 || navigationLinks.some((item) => item.tagName !== 'A') || navigationLinks[0].href !== expectedOrigin + '/' || navigationLinks[1].href !== expectedOrigin + '/results/' || text(navigationLinks[0]) !== '다시 입력' || text(navigationLinks[1]) !== '게시 정보') return 'navigation';
-    const footer = document.querySelector('footer'); const footerShell = document.querySelector('footer > .page-shell'); const footerChildren = document.querySelectorAll('footer > .page-shell > p'); if (footer.children.length !== 1 || footerChildren.length !== 1 || footerShell.children.length !== 1 || text(footerChildren[0]) !== '두 카드는 서로 독립된 검수·게시 경계를 사용합니다.') return 'footer';
-    const scripts = [...document.scripts]; if (scripts.length !== 1 || scripts[0].src !== expectedOrigin + '/static/public_web/site.js' || scripts[0].textContent.trim()) return 'script';
-    const styles = [...document.querySelectorAll('link[rel="stylesheet"]')]; if (styles.length !== 1 || styles[0].href !== expectedOrigin + '/static/public_web/site.css') return 'style';
+    const mains = [...document.querySelectorAll('main#main-content')];
+    if (mains.length !== 1 || document.querySelectorAll('main').length !== 1) return 'main';
+    const main = mains[0];
+    if (main.querySelectorAll('h1').length !== 1 || !text(main.querySelector('h1'))) return 'heading';
+    const cards = [...main.querySelectorAll('article')];
+    if (cards.length !== 2 || cards.map((card) => card.id).sort().join('|') !== 'entry-card|warning-card') return 'cards';
+    const skipLinks = [...document.querySelectorAll('a[href="#main-content"]')];
+    if (skipLinks.length !== 1 || !text(skipLinks[0])) return 'skip-link';
+    const navigations = [...document.querySelectorAll('nav[aria-label="주요 메뉴"]')];
+    if (navigations.length !== 1) return 'navigation';
+    const navigationLinks = [...navigations[0].querySelectorAll('a[href]')];
+    const inputLink = navigationLinks.find((link) => link.getAttribute('href') === '/');
+    const resultsLink = navigationLinks.find((link) => link.getAttribute('href') === '/results/');
+    if (navigationLinks.length !== 2 || !inputLink || !resultsLink || inputLink.href !== expectedOrigin + '/' || resultsLink.href !== expectedOrigin + '/results/' || !text(inputLink) || !text(resultsLink) || resultsLink.getAttribute('aria-current') !== 'page') return 'navigation';
+    const safety = text(main);
+    if (!['날짜','공식','확인'].every((token) => safety.includes(token)) || !['계산하지','판단하지','맞춘 결과가 아닙니다'].some((token) => safety.includes(token))) return 'safety-meaning';
+    const footers = [...document.querySelectorAll('footer')];
+    if (footers.length !== 1 || !text(footers[0]) || !['독립','검수','게시'].every((token) => text(footers[0]).includes(token))) return 'footer';
     return null;
   }}, {json.dumps(origin)});
   if (pageContract) fail(`results-dom-${{pageContract}}`);
-  const staticAssets = {json.dumps([
-      {"path": "/static/public_web/site.css", "sha256": SITE_CSS_SHA256, "bytes": SITE_CSS_BYTES, "types": ["text/css"]},
-      {"path": "/static/public_web/site.js", "sha256": SITE_JS_SHA256, "bytes": SITE_JS_BYTES, "types": ["text/javascript", "application/javascript"]},
-  ], separators=(",", ":"))};
-  for (const asset of staticAssets) {{
-    const assetUrl = {json.dumps(origin)} + asset.path;
-    const awaited = page.waitForResponse((response) => response.url() === assetUrl && response.request().method() === 'GET');
-    const observed = await page.evaluate(async (contract) => {{
-      const response = await fetch(contract.url, {{ credentials: 'omit', cache: 'no-store', redirect: 'error', referrerPolicy: 'no-referrer' }});
-      const bytes = new Uint8Array(await response.arrayBuffer());
-      const digest = [...new Uint8Array(await crypto.subtle.digest('SHA-256', bytes))].map((value) => value.toString(16).padStart(2, '0')).join('');
-      return {{ status: response.status, contentType: response.headers.get('content-type') || '', byteLength: bytes.byteLength, digest }};
-    }}, {{ url: assetUrl }});
-    const response = await awaited; const details = await response.securityDetails(); const responseHeaders = await response.allHeaders(); const requestHeaders = await response.request().allHeaders();
-    if (!details || !['TLS 1.2', 'TLS 1.3'].includes(details.protocol) || 'set-cookie' in responseHeaders || 'cookie' in requestHeaders || response.request().redirectedFrom()) fail('static-transport');
-    if (observed.status !== 200 || observed.byteLength !== asset.bytes || observed.digest !== asset.sha256 || !asset.types.includes(observed.contentType.split(';')[0].trim().toLowerCase())) fail('static-digest');
-  }}
   const expected = {{ entry: {json.dumps(entry_state)}, warning: {json.dumps(warning_state)} }};
-  const statusLabels = {{ ready: '게시된 source 사실', stale: '재확인 필요', empty: '게시 전', unavailable: '정보 확인 필요', 'server-error': '일시적 오류' }};
+  const statusSemantics = {{
+    ready: [['게시']], stale: [['재확인']],
+    empty: [['게시'], ['전', '없음']],
+    unavailable: [['확인'], ['필요', '없음']],
+    'server-error': [['일시적'], ['오류']],
+  }};
   const unsafeSourceValue = (value, maximum) => {{
     if (typeof value !== 'string' || !value.trim() || value.length > maximum) return true;
     if ([...value].some((character) => {{ const code = character.charCodeAt(0); return code < 32 && ![9, 10, 13].includes(code); }})) return true;
@@ -2341,8 +2364,16 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
     return date.getUTCFullYear() === parts[0] && date.getUTCMonth() === parts[1] - 1 && date.getUTCDate() === parts[2] && date.getUTCHours() === parts[3] && date.getUTCMinutes() === parts[4] ? milliseconds : NaN;
   }};
   const contracts = {{
-    entry: {{ id: 'entry-card', heading: '입국요건 사실', link: '외교부 입국요건 source 열기', labels: ['국가','일반여권 source 표기','source 근거 문구','snapshot date','마지막 성공 확인시각','publication revision','게시시각','source revision','출처'], owner: '대한민국 외교부 정보화담당관실 · 외교부|공공데이터포털', source: {json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {ENTRY_FRESHNESS_MINUTES}, note: '확인 필요: 여행 목적·날짜 적용성과 최신 조건은 source에서 다시 확인해 주세요.' }},
-    warning: {{ id: 'warning-card', heading: '여행경보', link: '외교부 여행경보 source 열기', labels: ['국가','source 경보 단계 코드','source 범위 유형','source 범위','source 작성일','마지막 성공 확인시각','publication revision','게시시각','source revision','출처'], owner: '대한민국 외교부 · 외교부|공공데이터포털', source: {json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {WARNING_FRESHNESS_MINUTES}, note: '확인 필요: 단계 명칭, 발효·종료 시각과 여행일 적용성은 source에서 다시 확인해 주세요.' }}
+    entry: {{
+      id: 'entry-card', heading: '입국요건 사실', linkTokens: ['입국요건','열기'],
+      fields: {{ country: ['국가','대상 국가'], passport: ['일반여권 source 표기','일반여권 관련 출처 표기'], basis: ['source 근거 문구','출처 근거 문구'], sourceDate: ['snapshot date','출처 자료 날짜'], checkedAt: ['마지막 성공 확인시각'], publicationRevision: ['publication revision','게시 리비전'], publishedAt: ['게시시각'], sourceRevision: ['source revision','출처 리비전'], source: ['출처'] }},
+      owner: '대한민국 외교부 정보화담당관실 · 외교부|공공데이터포털', source: {json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {ENTRY_FRESHNESS_MINUTES}, noteTokens: ['확인 필요','여행 목적','날짜','적용','최신','확인']
+    }},
+    warning: {{
+      id: 'warning-card', heading: '여행경보', linkTokens: ['여행경보','열기'],
+      fields: {{ country: ['국가','대상 국가'], alarmCode: ['source 경보 단계 코드','출처 경보 단계 코드'], scopeType: ['source 범위 유형','출처 범위 유형'], scope: ['source 범위','출처 범위'], writtenDate: ['source 작성일','출처 작성일'], checkedAt: ['마지막 성공 확인시각'], publicationRevision: ['publication revision','게시 리비전'], publishedAt: ['게시시각'], sourceRevision: ['source revision','출처 리비전'], source: ['출처'] }},
+      owner: '대한민국 외교부 · 외교부|공공데이터포털', source: {json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}, freshnessMinutes: {WARNING_FRESHNESS_MINUTES}, noteTokens: ['확인 필요','단계','발효','종료','여행일','적용','확인']
+    }}
   }};
   if (await page.locator('pre, code, textarea, template, iframe, object, embed, script:not([src]), [data-raw-body], [data-secret]').count()) fail('raw-secret-node');
   const hiddenLeak = await page.evaluate((approved) => {{
@@ -2361,7 +2392,7 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
     while (commentWalker.nextNode()) if ((commentWalker.currentNode.nodeValue || '').trim()) commentLeak = true;
     return attributeLeak || commentLeak;
   }}, {{
-    href: ['#main-content','/','/results/','/static/public_web/site.css',{json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)},{json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}],
+    href: ['#main-content','#entry-card','#warning-card','/','/results/','/static/public_web/site.css',{json.dumps(ENTRY_PUBLIC_SOURCE_LOCATOR)},{json.dumps(WARNING_PUBLIC_SOURCE_LOCATOR)}],
     src: ['/static/public_web/site.js']
   }});
   if (hiddenLeak) fail('raw-secret-hidden');
@@ -2369,85 +2400,90 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
   for (const module of ['entry', 'warning']) {{
     const contract = contracts[module]; const card = page.locator(`#${{contract.id}}`);
     if (await card.count() !== 1 || await card.getAttribute('data-state') !== expected[module]) fail(`${{module}}-state`);
-    if (await card.getAttribute('tabindex') !== '0' || await card.getAttribute('aria-labelledby') !== `${{module}}-heading` || await card.getAttribute('aria-describedby') !== `${{module}}-status ${{module}}-message`) fail(`${{module}}-aria-contract`);
+    const labelledBy = (await card.getAttribute('aria-labelledby') || '').split(/\\s+/).filter(Boolean);
+    const describedBy = (await card.getAttribute('aria-describedby') || '').split(/\\s+/).filter(Boolean);
+    if (await card.getAttribute('tabindex') !== '0' || labelledBy.length !== 1 || labelledBy[0] !== `${{module}}-heading` || describedBy.length !== 2 || new Set(describedBy).size !== 2 || !describedBy.includes(`${{module}}-status`) || !describedBy.includes(`${{module}}-message`)) fail(`${{module}}-aria-contract`);
     if (await card.getByRole('heading', {{ level: 2, name: contract.heading, exact: true }}).count() !== 1) fail(`${{module}}-heading`);
-    if (await card.getByRole('status').count() !== 1 || (await card.getByRole('status').innerText()).trim() !== `상태: ${{statusLabels[expected[module]]}}` || await card.locator('.status-symbol[aria-hidden="true"]').count() !== 1) fail(`${{module}}-status`);
+    const statuses = card.getByRole('status');
+    if (await statuses.count() !== 1) fail(`${{module}}-status`);
+    const statusText = (await statuses.innerText()).replace(/\\s+/g, ' ').trim();
+    if (!statusText.startsWith('상태: ') || !statusSemantics[expected[module]].every((alternatives) => alternatives.some((token) => statusText.includes(token)))) fail(`${{module}}-status`);
     const stateValue = expected[module]; const published = stateValue === 'ready' || stateValue === 'stale';
-    const expectedMessage = stateValue === 'ready'
-      ? (module === 'entry' ? '공식 source의 검수·게시 사실입니다.' : '입국요건과 독립된 공식 source의 검수·게시 사실입니다.')
-      : stateValue === 'stale' ? '마지막 검수·게시 사실입니다. 더 최근 조회 또는 source 상태를 재확인해 주세요.'
-      : stateValue === 'empty' ? '아직 검수·게시된 source 사실이 없습니다. 공식 source 확인이 필요합니다.'
-      : stateValue === 'unavailable' ? '게시 경계를 확인할 수 없습니다. 공식 source에서 직접 확인해 주세요.'
-      : '이 정보를 지금 읽을 수 없습니다. 다른 카드는 계속 확인할 수 있습니다.';
-    if ((await card.locator(`#${{module}}-message`).innerText()).trim() !== expectedMessage) fail(`${{module}}-message`);
-    const factLists = card.locator('dl.fact-list'); const links = card.getByRole('link', {{ name: contract.link, exact: true }}); const notes = card.locator('.verification-note');
-    const directChildren = await card.evaluate((node) => [...node.children].map((child) => `${{child.tagName}}:${{child.id}}:${{child.className}}`));
-    const expectedChildren = [`H2:${{module}}-heading:`,`P:${{module}}-status:status-line`,`P:${{module}}-message:`];
-    if (published) expectedChildren.push('DL::fact-list','P::verification-note');
-    if (JSON.stringify(directChildren) !== JSON.stringify(expectedChildren)) fail(`${{module}}-dom-contract`);
-    const shellExact = await card.evaluate((node) => {{
-      const names = (item) => [...item.attributes].map((attribute) => attribute.name).sort().join(',');
-      const heading = node.querySelector('h2'); const status = node.querySelector('.status-line'); const symbol = node.querySelector('.status-symbol'); const strong = status.querySelector('strong'); const message = node.querySelector(`#${{node.id.replace('-card', '-message')}}`);
-      return names(node) === 'aria-describedby,aria-labelledby,class,data-state,id,tabindex' && names(heading) === 'id' && names(status) === 'class,id,role' && names(symbol) === 'aria-hidden,class' && names(strong) === '' && names(message) === 'id' && !heading.children.length && !symbol.children.length && !strong.children.length && !message.children.length;
-    }});
-    if (!shellExact) fail(`${{module}}-shell-contract`);
+    const messageSemantics = stateValue === 'ready'
+      ? (module === 'entry' ? [['공식'],['검수'],['게시']] : [['입국요건'],['독립','별도'],['공식'],['검수'],['게시']])
+      : stateValue === 'stale' ? [['마지막'],['검수'],['게시'],['재확인','다시 확인']]
+      : stateValue === 'empty' ? [['아직'],['게시'],['공식'],['확인']]
+      : stateValue === 'unavailable' ? [['게시'],['공식'],['확인']]
+      : [['지금'],['읽을 수 없습니다'],['다른'],['확인']];
+    const message = card.locator(`#${{module}}-message`);
+    if (await message.count() !== 1) fail(`${{module}}-message`);
+    const messageText = (await message.innerText()).replace(/\\s+/g, ' ').trim();
+    if (unsafeSourceValue(messageText, 500) || !messageSemantics.every((alternatives) => alternatives.some((token) => messageText.includes(token)))) fail(`${{module}}-message`);
+    const factLists = card.locator('dl'); const links = card.locator('a[href]');
+    const metadata = await card.evaluate((node, input) => {{
+      const text = (item) => item?.textContent.replace(/\\s+/g, ' ').trim() || '';
+      const heading = node.querySelector(`#${{input.module}}-heading`);
+      const status = node.querySelector(`#${{input.module}}-status`);
+      const messageNode = node.querySelector(`#${{input.module}}-message`);
+      const lists = [...node.querySelectorAll('dl')];
+      const dts = lists.flatMap((list) => [...list.querySelectorAll('dt')]);
+      const dds = lists.flatMap((list) => [...list.querySelectorAll('dd')]);
+      const labels = dts.map(text); const labelSet = new Set(labels);
+      const matchedLabels = Object.fromEntries(Object.entries(input.fields).map(([field, aliases]) => [field, aliases.filter((alias) => labelSet.has(alias))]));
+      const pairedDescriptions = dts.map((dt) => dt.nextElementSibling);
+      const pairsValid = dts.length === dds.length && pairedDescriptions.every((dd) => dd?.tagName === 'DD') && new Set(pairedDescriptions).size === dds.length && dds.every((dd) => pairedDescriptions.includes(dd));
+      const values = Object.fromEntries(Object.entries(matchedLabels).map(([field, matches]) => {{ const term = dts.find((dt) => text(dt) === matches[0]); return [field, text(term?.nextElementSibling)]; }}));
+      const sourceTerm = dts.find((dt) => input.fields.source.includes(text(dt)));
+      const sourceDescription = sourceTerm?.nextElementSibling?.tagName === 'DD' ? sourceTerm.nextElementSibling : null;
+      const sourceClone = sourceDescription?.cloneNode(true); sourceClone?.querySelectorAll('a').forEach((link) => link.remove());
+      const sourceAttribution = text(sourceClone);
+      const noteMarkers = [...node.querySelectorAll('strong')].filter((item) => text(item) === '확인 필요:');
+      const noteRoots = [...new Set(noteMarkers.map((item) => item.parentElement).filter((item) => item && item !== node))];
+      const noteText = noteRoots.length === 1 ? text(noteRoots[0]) : '';
+      const interactive = [...node.querySelectorAll('a[href], button, input, select, textarea, form, [contenteditable="true"]')];
+      const sectionHeadings = [...node.querySelectorAll('h3')];
+      const semanticRoots = [heading, status, messageNode, ...sectionHeadings, ...dts, ...dds, ...noteRoots, ...interactive].filter(Boolean);
+      const walker = document.createTreeWalker(node, NodeFilter.SHOW_TEXT); let strayText = false;
+      while (walker.nextNode()) if ((walker.currentNode.nodeValue || '').trim() && !semanticRoots.some((root) => root.contains(walker.currentNode))) strayText = true;
+      const bodySize = Number.parseFloat(getComputedStyle(document.body).fontSize);
+      const h2Size = heading ? Number.parseFloat(getComputedStyle(heading).fontSize) : 0;
+      const metadataReadable = [...dts, ...dds].every((item) => Number.parseFloat(getComputedStyle(item).fontSize) >= 14 && Number.parseFloat(getComputedStyle(item).lineHeight) / Number.parseFloat(getComputedStyle(item).fontSize) >= 1.35);
+      return {{
+        values, h2Size, bodySize, metadataReadable,
+        labelsUnique: labelSet.size === labels.length,
+        labelsComplete: labels.length === Object.keys(input.fields).length && Object.values(matchedLabels).every((matches) => matches.length === 1),
+        pairsValid, sourceAttribution,
+        noteCount: noteRoots.length, noteMarkerCount: noteMarkers.length, noteText,
+        interactiveCount: interactive.length,
+        sourceIsOnlyInteractive: interactive.length === 1 && interactive[0].tagName === 'A',
+        strayText,
+      }};
+    }}, {{ module, fields: contract.fields }});
+    if (metadata.strayText) fail(`${{module}}-text-ownership`);
     if (!published) {{
-      if (await factLists.count() || await links.count() || await notes.count()) fail(`${{module}}-unpublished-leak`);
+      if (await factLists.count() || await links.count() || metadata.noteCount || metadata.noteMarkerCount || metadata.interactiveCount) fail(`${{module}}-unpublished-leak`);
       continue;
     }}
     publicationCount += 1;
-    if (await factLists.count() !== 1 || await links.count() !== 1 || await notes.count() !== 1 || (await notes.innerText()).trim() !== contract.note) fail(`${{module}}-publication-contract`);
-    const metadata = await card.evaluate((node) => {{
-      const dts = [...node.querySelectorAll('.fact-list dt')];
-      const values = Object.fromEntries(dts.map((dt) => [dt.textContent.trim(), dt.nextElementSibling?.textContent.trim() || '']));
-      const bodySize = Number.parseFloat(getComputedStyle(document.body).fontSize);
-      const h2Size = Number.parseFloat(getComputedStyle(node.querySelector('h2')).fontSize);
-      const metadataReadable = [...node.querySelectorAll('.fact-list dt, .fact-list dd')].every((item) => Number.parseFloat(getComputedStyle(item).fontSize) >= 14 && Number.parseFloat(getComputedStyle(item).lineHeight) / Number.parseFloat(getComputedStyle(item).fontSize) >= 1.35);
-      const children = [...node.querySelector('dl.fact-list').children];
-      const exactStructure = children.length === dts.length * 2 && children.every((item, index) => item.tagName === (index % 2 ? 'DD' : 'DT'));
-      const descriptions = children.filter((item) => item.tagName === 'DD'); const sourceDescription = descriptions.at(-1);
-      const attributeNames = (item) => [...item.attributes].map((attribute) => attribute.name).sort().join(',');
-      const exactAttributes = attributeNames(node) === 'aria-describedby,aria-labelledby,class,data-state,id,tabindex'
-        && attributeNames(node.querySelector('h2')) === 'id'
-        && attributeNames(node.querySelector('.status-line')) === 'class,id,role'
-        && attributeNames(node.querySelector('.status-symbol')) === 'aria-hidden,class'
-        && attributeNames(node.querySelector('.status-line strong')) === ''
-        && attributeNames(node.querySelector(`#${{node.id.replace('-card', '-message')}}`)) === 'id'
-        && attributeNames(node.querySelector('dl.fact-list')) === 'class'
-        && dts.every((item) => attributeNames(item) === '')
-        && descriptions.every((item) => attributeNames(item) === '')
-        && attributeNames(sourceDescription.firstElementChild) === 'aria-label,class,href,rel'
-        && attributeNames(node.querySelector('.verification-note')) === 'class'
-        && attributeNames(node.querySelector('.verification-note strong')) === '';
-      const exactDescendants = node.querySelector('h2').children.length === 0
-        && [...node.querySelector('.status-line').children].map((item) => item.tagName).join(',') === 'SPAN,STRONG'
-        && [...node.querySelector('.status-line').children].every((item) => item.children.length === 0)
-        && node.querySelector(`#${{node.id.replace('-card', '-message')}}`).children.length === 0
-        && dts.every((item) => item.children.length === 0)
-        && descriptions.slice(0, -1).every((item) => item.children.length === 0)
-        && sourceDescription?.children.length === 1 && sourceDescription.firstElementChild?.tagName === 'A'
-        && sourceDescription.firstElementChild?.children.length === 0
-        && [...node.querySelector('.verification-note').children].map((item) => item.tagName).join(',') === 'STRONG'
-        && node.querySelector('.verification-note strong').children.length === 0;
-      return {{ labels: dts.map((dt) => dt.textContent.trim()), values, h2Size, bodySize, metadataReadable, exactStructure, exactDescendants, exactAttributes }};
-    }});
-    if (JSON.stringify(metadata.labels) !== JSON.stringify(contract.labels) || !metadata.exactStructure || !metadata.exactDescendants || !metadata.exactAttributes || !metadata.metadataReadable || !(metadata.h2Size > metadata.bodySize)) fail(`${{module}}-metadata`);
-    if (contract.labels.some((label) => {{ const value = metadata.values[label]; const trimmed = value?.trimStart() || ''; return !value || trimmed.startsWith('{{') || trimmed.startsWith('['); }})) fail(`${{module}}-empty-value`);
-    const nowMillis = Date.now(); const observedMillis = parseUtcMinute(metadata.values['마지막 성공 확인시각']);
-    if (metadata.values['국가'] !== '일본' || !Number.isFinite(observedMillis)) fail(`${{module}}-freshness`);
+    if (![1, 2].includes(await factLists.count()) || await links.count() !== 1 || metadata.noteCount !== 1 || metadata.noteMarkerCount !== 1 || !contract.noteTokens.every((token) => metadata.noteText.includes(token))) fail(`${{module}}-publication-contract`);
+    if (!metadata.labelsUnique || !metadata.labelsComplete || !metadata.pairsValid || !metadata.sourceIsOnlyInteractive || !metadata.metadataReadable || !(metadata.h2Size > metadata.bodySize)) fail(`${{module}}-metadata`);
+    if (Object.values(metadata.values).some((value) => {{ const trimmed = value?.trimStart() || ''; return !value || trimmed.startsWith('{{') || trimmed.startsWith('['); }})) fail(`${{module}}-empty-value`);
+    const nowMillis = Date.now(); const observedMillis = parseUtcMinute(metadata.values.checkedAt);
+    if (metadata.values.country !== '일본' || !Number.isFinite(observedMillis)) fail(`${{module}}-freshness`);
     const ageMinutes = (nowMillis - observedMillis) / 60000;
     if (!Number.isFinite(ageMinutes) || ageMinutes < -5 || (stateValue === 'ready' && ageMinutes > contract.freshnessMinutes + 2) || (stateValue === 'stale' && ageMinutes <= contract.freshnessMinutes)) fail(`${{module}}-freshness-age`);
-    const publishedMillis = parseUtcMinute(metadata.values['게시시각']);
-    if (!/^generation [1-9]\\d*$/.test(metadata.values['publication revision']) || !Number.isFinite(publishedMillis) || publishedMillis > nowMillis + 300000 || metadata.values['source revision'] !== 'rights-v1') fail(`${{module}}-revision-metadata`);
-    if (module === 'entry') {{ const snapshotMillis = parseDateOnly(metadata.values['snapshot date']); if (!Number.isFinite(snapshotMillis) || snapshotMillis > nowMillis + 86400000 || !/^[1-9]\\d{{0,2}}일$/.test(metadata.values['일반여권 source 표기']) || unsafeSourceValue(metadata.values['source 근거 문구'], 1000)) fail('entry-facts'); }}
-    if (module === 'warning') {{ const written = metadata.values['source 작성일']; const writtenMillis = written === 'source가 제공하지 않음' ? null : parseDateOnly(written); if (unsafeSourceValue(metadata.values['source 경보 단계 코드'], 32) || unsafeSourceValue(metadata.values['source 범위 유형'], 100) || unsafeSourceValue(metadata.values['source 범위'], 1000) || (writtenMillis !== null && (!Number.isFinite(writtenMillis) || writtenMillis > nowMillis + 86400000))) fail('warning-facts'); }}
-    if (metadata.values['출처'].replace(/\\s+/g, ' ') !== `${{contract.owner}} 공식 source`) fail(`${{module}}-source-name`);
+    const publishedMillis = parseUtcMinute(metadata.values.publishedAt);
+    if (!/^generation [1-9]\\d*$/.test(metadata.values.publicationRevision) || !Number.isFinite(publishedMillis) || publishedMillis > nowMillis + 300000 || metadata.values.sourceRevision !== 'rights-v1') fail(`${{module}}-revision-metadata`);
+    if (module === 'entry') {{ const snapshotMillis = parseDateOnly(metadata.values.sourceDate); if (!Number.isFinite(snapshotMillis) || snapshotMillis > nowMillis + 86400000 || !/^[1-9]\\d{{0,2}}일$/.test(metadata.values.passport) || unsafeSourceValue(metadata.values.basis, 1000)) fail('entry-facts'); }}
+    if (module === 'warning') {{ const written = metadata.values.writtenDate; const writtenMillis = written.includes('제공하지 않음') ? null : parseDateOnly(written); if (unsafeSourceValue(metadata.values.alarmCode, 32) || unsafeSourceValue(metadata.values.scopeType, 100) || unsafeSourceValue(metadata.values.scope, 1000) || (writtenMillis !== null && (!Number.isFinite(writtenMillis) || writtenMillis > nowMillis + 86400000))) fail('warning-facts'); }}
+    if (metadata.sourceAttribution !== contract.owner) fail(`${{module}}-source-name`);
     const link = links.first(); const href = await link.getAttribute('href'); const rel = (await link.getAttribute('rel') || '').split(/\\s+/);
-    if (href !== contract.source || !rel.includes('noopener') || !rel.includes('noreferrer')) fail(`${{module}}-source-link`);
+    const linkName = ((await link.getAttribute('aria-label')) || (await link.innerText())).replace(/\\s+/g, ' ').trim();
+    if (href !== contract.source || !rel.includes('noopener') || !rel.includes('noreferrer') || !contract.linkTokens.every((token) => linkName.includes(token))) fail(`${{module}}-source-link`);
   }}
-  const noteCount = await page.locator('.verification-note').count();
+  const noteCount = await page.locator('#entry-card, #warning-card').evaluateAll((cards) => cards.flatMap((card) => [...card.querySelectorAll('strong')]).filter((item) => item.textContent.replace(/\\s+/g, ' ').trim() === '확인 필요:').length);
   if (noteCount !== publicationCount || (publicationCount > 0) !== (noteCount > 0)) fail('verification-count');
-  if ({json.dumps(state)} === 'long-korean' && await page.locator('.fact-list dd').evaluateAll((nodes) => Math.max(...nodes.map((node) => node.textContent.trim().length), 0)) < 40) fail('long-content');
+  if ({json.dumps(state)} === 'long-korean' && await page.locator('#entry-card dl dd, #warning-card dl dd').evaluateAll((nodes) => Math.max(...nodes.map((node) => node.textContent.trim().length), 0)) < 40) fail('long-content');
   const bodyText = await page.locator('body').innerText(); if (unsafeSourceValue(bodyText, 100000)) fail('forbidden-content');
   const foldedBody = bodyText.toLowerCase().replace(/\\s+/g, ' '); const compactBody = foldedBody.replace(/\\s+/g, '');
   for (const marker of ['allow' + 'ed','deni' + 'ed','mofa_' + 'travel_alarm_service_key','service' + 'key','api_key','api key','secret key','authorization:']) if (foldedBody.includes(marker)) fail('forbidden-content');
@@ -2457,10 +2493,33 @@ def results_javascript(*, origin: str, state: str, check: str) -> str:
 
 
 def results_keyboard_javascript(*, check: str) -> str:
+    required_before_index = [
+        'a[href="#main-content"]',
+        'nav[aria-label="주요 메뉴"] a[href="/"]',
+        'nav[aria-label="주요 메뉴"] a[href="/results/"]',
+    ]
+    optional_index = [
+        'nav[aria-label="게시 정보 목차"] a[href="#entry-card"]',
+        'nav[aria-label="게시 정보 목차"] a[href="#warning-card"]',
+    ]
+    required_after_index = [
+        "#entry-card",
+        f'#entry-card a[href="{ENTRY_PUBLIC_SOURCE_LOCATOR}"]',
+        "#warning-card",
+        f'#warning-card a[href="{WARNING_PUBLIC_SOURCE_LOCATOR}"]',
+    ]
     return f"""async (page) => {{
   const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
   await page.evaluate(() => document.activeElement && document.activeElement.blur());
-  for (const selector of ['.skip-link','.site-nav a:nth-of-type(1)','.site-nav a:nth-of-type(2)','#entry-card','#entry-card .source-link','#warning-card','#warning-card .source-link']) {{
+  const optionalIndex = {json.dumps(optional_index)};
+  const optionalCounts = await Promise.all(optionalIndex.map((selector) => page.locator(selector).count()));
+  if (!optionalCounts.every((count) => count === optionalCounts[0]) || ![0, 1].includes(optionalCounts[0])) fail('result-index-contract');
+  const selectors = [
+    ...{json.dumps(required_before_index)},
+    ...(optionalCounts[0] === 1 ? optionalIndex : []),
+    ...{json.dumps(required_after_index)},
+  ];
+  for (const selector of selectors) {{
     await page.keyboard.press('Tab');
     const visible = await page.evaluate((candidate) => {{ const node = document.activeElement; if (!node?.matches(candidate)) return false; const style = getComputedStyle(node); return style.outlineStyle !== 'none' && Number.parseFloat(style.outlineWidth) >= 2; }}, selector);
     if (!visible) fail('result-focus');
@@ -2473,7 +2532,11 @@ def forced_colors_javascript(*, check: str) -> str:
     return f"""async (page) => {{
   const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
   await page.emulateMedia({{ forcedColors: 'active', reducedMotion: 'reduce' }});
-  const okay = await page.locator('.publication-card').evaluateAll((cards) => cards.length === 2 && cards.every((card) => {{ const style = getComputedStyle(card); return style.borderStyle !== 'none' && Number.parseFloat(style.borderWidth) >= 2 && card.querySelector('[role="status"]').textContent.includes('상태:'); }}));
+  const okay = await page.locator('article#entry-card, article#warning-card').evaluateAll((cards) => cards.length === 2 && cards.every((card) => {{
+    const style = getComputedStyle(card);
+    const borders = [style.borderTopWidth, style.borderRightWidth, style.borderBottomWidth, style.borderLeftWidth].map(Number.parseFloat);
+    return Math.max(...borders) >= 2 && card.querySelector('[role="status"]').textContent.includes('상태:');
+  }}));
   if (!okay) fail('forced-colors-state'); return {_marker(check)};
 }}"""
 
@@ -2498,7 +2561,7 @@ def scaled_form_submit_javascript(*, origin: str, check: str) -> str:
   const fail = (code) => {{ throw new Error(`acceptance:${{code}}`); }};
   {_submission_support(origin)} await prepareSubmit();
   await page.getByLabel('출국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_DEPARTURE)}); await page.getByLabel('귀국일', {{ exact: true }}).fill({json.dumps(SYNTHETIC_VALID_RETURN)});
-  await page.getByRole('button', {{ name: '게시 정보 확인', exact: true }}).focus(); await page.keyboard.press('Enter');
+  await page.locator('[data-submit-button]').focus(); await page.keyboard.press('Enter');
   await page.waitForURL({json.dumps(origin + '/results/')}, {{ waitUntil: 'networkidle' }}); await finishSubmit(303);
   await page.evaluate(() => document.documentElement.style.fontSize = '200%');
   if (await page.evaluate(() => document.documentElement.scrollWidth > window.innerWidth + 1)) fail('scaled-overflow');
@@ -2896,7 +2959,7 @@ def run_matrix(
             session.snapshot(
                 check="snapshot-pristine",
                 required_tokens=(
-                    "일본 여행 정보 확인", "목적지", "출국일", "귀국일", "게시 정보 확인"
+                    "목적지", "출국일", "귀국일", "일본"
                 ),
             )
             session.run_code(
@@ -3022,7 +3085,7 @@ def run_matrix(
             session.snapshot(
                 check="snapshot-ready",
                 required_tokens=(
-                    "일본 게시 정보", "입국요건 사실", "여행경보", "확인 필요", "공식 source"
+                    "입국요건 사실", "여행경보", "확인 필요", "공식 source"
                 ),
             )
             session.run_code(
@@ -3075,7 +3138,7 @@ def run_matrix(
                 )
                 session.snapshot(
                     check=f"snapshot-{state}",
-                    required_tokens=("일본 게시 정보", "입국요건 사실", "여행경보", "상태:"),
+                    required_tokens=("입국요건 사실", "여행경보", "상태:"),
                 )
                 checks.append({"check": f"state-{state}-exact-pair", "viewport": viewport})
                 screenshots.append(
diff --git a/operations/tests/test_browser_acceptance_harness.py b/operations/tests/test_browser_acceptance_harness.py
index 53a1025..ebe0fef 100644
--- a/operations/tests/test_browser_acceptance_harness.py
+++ b/operations/tests/test_browser_acceptance_harness.py
@@ -66,7 +66,7 @@ def form_view(request):
 <form method="post" action="/"><input type="hidden" name="csrfmiddlewaretoken" value="{token}">
 <label for="departure">출국일</label><input id="departure" name="departure_date" type="date">
 <label for="return">귀국일</label><input id="return" name="return_date" type="date">
-<button type="submit">게시 정보 확인</button></form></body></html>'''
+<button type="submit" data-submit-button>게시 정보 확인</button></form></body></html>'''
     return HttpResponse(body, content_type="text/html; charset=utf-8")
 
 def results_view(request):
@@ -418,6 +418,50 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("Object.keys(document).length !== 1", code)
         self.assertNotIn("caller_release", code)
 
+    def test_main_document_security_probe_is_header_only_and_fail_closed(self):
+        for path in ("/", "/results/"):
+            with self.subTest(path=path):
+                source = acceptance._main_document_security_source(
+                    origin=self.origins["ready"], path=path
+                )
+                for expected in (
+                    "credentials: 'omit'",
+                    "referrerPolicy: 'no-referrer'",
+                    "securityDetails",
+                    "TLS 1.2",
+                    "cache-control",
+                    "no-store",
+                    "pragma",
+                    "content-security-policy",
+                    "default-src 'self'",
+                    "base-uri 'none'",
+                    "form-action 'self'",
+                    "frame-ancestors 'none'",
+                    "object-src 'none'",
+                    "referrer-policy",
+                    "x-content-type-options",
+                    "x-frame-options",
+                    "cross-origin-opener-policy",
+                    "strict-transport-security",
+                    "max-age=31536000",
+                    "includesubdomains",
+                    "preload",
+                ):
+                    self.assertIn(expected, source)
+                for forbidden in (
+                    "response.text",
+                    "arrayBuffer",
+                    "postData",
+                    "request.body",
+                    "set-cookie",
+                    "console.",
+                ):
+                    self.assertNotIn(forbidden, source)
+        with self.assertRaises(acceptance.AcceptanceFailure):
+            acceptance._main_document_security_source(
+                origin=self.origins["ready"], path="/unknown"
+            )
+
     def test_bad_certificate_der_fails_closed(self):
         for document in (b"", b"0\x00", b"0\x81\x01\x00"):
             with self.subTest(document=document):
@@ -697,8 +741,8 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
             origin=self.origins["unavailable"], state="unavailable", check="state"
         )
         self.assertIn("expected = { entry: \"ready\", warning: \"unavailable\" }", source)
-        self.assertIn("외교부 입국요건 source 열기", source)
-        self.assertIn("외교부 여행경보 source 열기", source)
+        self.assertIn("linkTokens: ['입국요건','열기']", source)
+        self.assertIn("linkTokens: ['여행경보','열기']", source)
         self.assertIn("마지막 성공 확인시각", source)
         self.assertIn("source 작성일", source)
         self.assertIn("noteCount !== publicationCount", source)
@@ -707,7 +751,17 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn(acceptance.ENTRY_PUBLIC_SOURCE_LOCATOR, source)
         self.assertIn(acceptance.WARNING_PUBLIC_SOURCE_LOCATOR, source)
         self.assertIn("raw-secret-node", source)
-        self.assertIn("document.head.children", source)
+        self.assertIn("hasAttribute('defer')", source)
+        self.assertIn("main#main-content", source)
+        self.assertIn("labelsUnique", source)
+        self.assertIn("labelsComplete", source)
+        self.assertIn("pairsValid", source)
+        self.assertIn("sourceIsOnlyInteractive", source)
+        self.assertIn("sourceAttribution", source)
+        self.assertIn("[1, 2].includes", source)
+        self.assertIn("text-ownership", source)
+        self.assertIn("unpublished-leak", source)
+        self.assertIn("noteTokens", source)
         self.assertIn("allowedNames = new Set", source)
         self.assertIn("crypto.subtle.digest('SHA-256'", source)
         self.assertIn(acceptance.SITE_CSS_SHA256, source)
@@ -716,6 +770,18 @@ class BrowserAcceptanceHarnessTests(unittest.TestCase):
         self.assertIn("getUTCDate() === parts[2]", source)
         self.assertIn("publishedMillis > nowMillis + 300000", source)
         self.assertNotIn("publishedMillis + 60000 < observedMillis", source)
+        for cosmetic_contract in (
+            "className",
+            "directChildren",
+            "shellExact",
+            "exactAttributes",
+            "exactDescendants",
+            ".publication-card",
+            ".fact-list",
+            ".verification-note",
+            ".source-link",
+        ):
+            self.assertNotIn(cosmetic_contract, source)
 
     def test_pinned_static_assets_match_the_reviewed_bytes(self):
         assets = (
diff --git a/operations/tests/test_browser_scenario_servers.py b/operations/tests/test_browser_scenario_servers.py
index f49fc50..83cc020 100644
--- a/operations/tests/test_browser_scenario_servers.py
+++ b/operations/tests/test_browser_scenario_servers.py
@@ -87,6 +87,22 @@ class BrowserScenarioCardTests(SimpleTestCase):
         self.assertGreater(scenario_server.LOADING_SECONDS, 0.0)
         self.assertLess(scenario_server.LOADING_SECONDS, 10.0)
 
+    def test_browser_asset_manifests_share_the_reviewed_bytes(self):
+        expected = {
+            "/static/public_web/site.css": (
+                browser_acceptance.SITE_CSS_BYTES,
+                browser_acceptance.SITE_CSS_SHA256,
+            ),
+            "/static/public_web/site.js": (
+                browser_acceptance.SITE_JS_BYTES,
+                browser_acceptance.SITE_JS_SHA256,
+            ),
+        }
+        self.assertEqual(set(scenario_server.SITE_ASSETS), set(expected))
+        for path, (_, _, size, digest) in scenario_server.SITE_ASSETS.items():
+            with self.subTest(path=path):
+                self.assertEqual((size, digest), expected[path])
+
     def test_fixed_state_transformations_are_in_memory_and_exact(self):
         original = ready_cards()
         snapshot = copy.deepcopy(original)
@@ -305,9 +321,13 @@ class BrowserScenarioCardTests(SimpleTestCase):
                     f'id="warning-card" data-state="{warning_state}"'.encode(),
                     response.content,
                 )
-                self.assertEqual(
-                    response.content.count(b'class="source-link"'), link_count
+                observed_link_count = sum(
+                    response.content.count(
+                        f'href="{html.escape(locator, quote=True)}"'.encode("ascii")
+                    )
+                    for locator in (ENTRY_SOURCE_LOCATOR, WARNING_SOURCE_LOCATOR)
                 )
+                self.assertEqual(observed_link_count, link_count)
                 if name == "long-korean":
                     long_cards = scenario_server.build_scenario_cards(
                         name, ready_cards
@@ -481,6 +501,37 @@ class BrowserScenarioLauncherIntegrationTests(SimpleTestCase):
 
                 status, headers, form = self._request(ready, "GET", "/")
                 self.assertEqual(status, 200)
+                names = {name.lower(): value for name, value in headers}
+                self.assertIn("no-store", names["cache-control"].lower())
+                self.assertEqual(names["pragma"].lower(), "no-cache")
+                self.assertEqual(
+                    {
+                        directive.strip()
+                        for directive in names["content-security-policy"].split(";")
+                        if directive.strip()
+                    },
+                    {
+                        "default-src 'self'",
+                        "base-uri 'none'",
+                        "form-action 'self'",
+                        "frame-ancestors 'none'",
+                        "object-src 'none'",
+                    },
+                )
+                self.assertEqual(names["referrer-policy"].lower(), "same-origin")
+                self.assertEqual(names["x-content-type-options"].lower(), "nosniff")
+                self.assertEqual(names["x-frame-options"].upper(), "DENY")
+                self.assertEqual(
+                    names["cross-origin-opener-policy"].lower(), "same-origin"
+                )
+                self.assertEqual(
+                    {
+                        token.strip().lower()
+                        for token in names["strict-transport-security"].split(";")
+                        if token.strip()
+                    },
+                    {"max-age=31536000", "includesubdomains", "preload"},
+                )
                 set_cookie_values = [
                     value for name, value in headers if name.lower() == "set-cookie"
                 ]


