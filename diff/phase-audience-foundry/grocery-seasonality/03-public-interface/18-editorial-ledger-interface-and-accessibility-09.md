## `test(evidence): record frontend redesign acceptance`

diff --git a/output/playwright/vnext-redesign-v2/1440x900-catalog.png b/output/playwright/vnext-redesign-v2/1440x900-catalog.png
new file mode 100644
index 0000000..d288d2a
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/1440x900-catalog.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-detail.png b/output/playwright/vnext-redesign-v2/1440x900-detail.png
new file mode 100644
index 0000000..5f1f872
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/1440x900-detail.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-history.png b/output/playwright/vnext-redesign-v2/1440x900-history.png
new file mode 100644
index 0000000..c3132dd
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/1440x900-history.png differ
diff --git a/output/playwright/vnext-redesign-v2/1440x900-selection.png b/output/playwright/vnext-redesign-v2/1440x900-selection.png
new file mode 100644
index 0000000..ff4d560
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/1440x900-selection.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-catalog.png b/output/playwright/vnext-redesign-v2/390x844-catalog.png
new file mode 100644
index 0000000..29a7baf
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/390x844-catalog.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-detail.png b/output/playwright/vnext-redesign-v2/390x844-detail.png
new file mode 100644
index 0000000..4a5910d
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/390x844-detail.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-history.png b/output/playwright/vnext-redesign-v2/390x844-history.png
new file mode 100644
index 0000000..8c549c4
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/390x844-history.png differ
diff --git a/output/playwright/vnext-redesign-v2/390x844-selection.png b/output/playwright/vnext-redesign-v2/390x844-selection.png
new file mode 100644
index 0000000..59ea6fc
Binary files /dev/null and b/output/playwright/vnext-redesign-v2/390x844-selection.png differ
diff --git a/output/playwright/vnext-redesign-v2/README.md b/output/playwright/vnext-redesign-v2/README.md
new file mode 100644
index 0000000..84c7bc0
--- /dev/null
+++ b/output/playwright/vnext-redesign-v2/README.md
@@ -0,0 +1,79 @@
+# Frontend redesign v2 local browser evidence
+
+생성 시각은 `2026-08-31T07:50:19Z`이며 검증 대상 frontend candidate는
+`d97888885e8a2e5b8db88005ddf0bf3a336dcdc6`다. 기존 Phase 0와 vNext evidence는 수정하지
+않았다.
+
+이 matrix의 ready 화면은 synthetic fixture가 아니다. 승인된 로컬 실행에서 KAMIS 공공 API의
+최근·월별·지역별·시장별 자료를 가져와 기존 source adapter, typed persistence, 검토·seal·activation
+service를 통과시킨 뒤 같은 active publication을 Django SSR로 조회했다. raw response는 hash-only
+정책에 따라 보존하지 않았고 source credential, request query와 원응답은 evidence에 기록하지 않았다.
+
+fixture의 mapping과 approval은 browser acceptance용으로 자동 생성한 test publication이다. 사람이
+검토한 production publication이나 production activation 증거가 아니며, historical coverage는 catalog
+첫 품목과 한 지역의 대표 소비자 흐름으로 제한된다. 최근 catalog의 10개 항목은 모두 실제 API에서
+정규화한 공개 행이다.
+
+## 실행 결과
+
+- [live source receipt](live-source-results.json): recent 10행, monthly 36행, regional 1행,
+  market 9행을 정규화·test-publish하고 5개 SSR route에서 source 재호출 0을 확인했다.
+- [browser receipt](browser-results.json): catalog → detail → history → regions → markets → detail →
+  두 품목 selection의 no-JS 흐름을 `390×844`와 `1440×900`에서 완료했다.
+- `390×844` catalog의 첫 record는 `y=635.80`, `height=156.53`으로 scroll 전 viewport 안에
+  완전히 표시됐다.
+- 같은 6개 surface를 `360×800`, `768×1024`에서 확인했고 horizontal overflow는 0이었다.
+- 모든 문서는 한 개의 `main`·`h1`, 44px 이상 target, 자연스러운 keyboard order, visible
+  skip-link와 item-link focus, client script 0, inline handler 0, 외부 asset/request 0을 유지했다.
+- no-store, no-referrer, `script-src 'none'`, nosniff, frame deny, cookie 없음과 recent/historical
+  fact-set header 분리를 확인했다.
+- [axe receipt](axe-results.json): local axe-core 4.13.0으로 ready 6면, validation 400, catalog
+  server-error 503, generic 404를 검사해 WCAG 2/2.1/2.2 A·AA violation 0과 unexpected
+  incomplete 0을 확인했다. 자동 판정이 불가능한 decorative symbol과 SVG label contrast는
+  별도 palette gate의 4.5:1 검사를 통과했다.
+
+## 도구
+
+- Playwright CLI `0.1.18`, Playwright core `1.63.0-alpha-2026-08-05`
+- Google Chrome `151.0.7922.174`, Node.js `23.11.0`
+- local axe-core `4.13.0`, SHA-256
+  `c24f097bd2f451d4f933e8bc7d8d539f8672a2ebcb5cc9f9f3eec8ca9470a0c1`
+- browser acceptance script SHA-256
+  `84680477b653b2c504b3f3dc8459bd201dd405f5d9e7ee27cf13b52de6f9ec12`
+- axe acceptance script SHA-256
+  `721fa8dc645c5b58ff1b791a166cad0e9b595feb1988b836a1990e928669cb14`
+
+## 통합 디자인 리뷰
+
+- Product: category 탐색과 실제 조사값 장부가 보조 검색보다 앞서며 mobile 첫 결과가 fold 안에
+  들어온다. 선택 화면은 결과를 편집 control보다 먼저 보여준다.
+- Copy: KAMIS 제공 사실, 조사 조건, 날짜, 개별 판매처 금액이 아니라는 caveat를 유지하고 내부
+  source·series·응답 용어를 노출하지 않는다.
+- Brand/UI: warm paper, forest/harvest/data-blue와 굵은 장부선을 사용하되 photo, gradient,
+  glass, dashboard card grid를 사용하지 않는다. 기능 제목은 system sans, 브랜드와 큰 editorial
+  제목만 Gowun Batang이다.
+- Engineering: Django SSR, no-JS, active-publication-only read, bounded query, template 산술 금지,
+  fail-closed validation과 보안 header를 유지한다.
+
+첫 실제 render 리뷰에서 dark masthead focus/hover 대비, historical 503의 빈 제목과 잘못된 복구
+label, detail의 market 탐색 단서, desktop catalog 열 배치, SVG 끝 월 label clipping을 수정했다.
+axe가 찾은 generic `div`의 unsupported `aria-label`도 제거한 뒤 전체 matrix를 다시 생성했다.
+
+## 파일 무결성
+
+| 파일 | SHA-256 |
+|---|---|
+| `390x844-catalog.png` | `193f09f97826858b2ce6da28768805b73ccbdb9501160b3c4e615c9e4b9a339b` |
+| `390x844-detail.png` | `ffce845e16d3ebbb5727ed4b16f0c2f4194612c7b30a8e2c7a90a1138cf6712d` |
+| `390x844-history.png` | `8eb3d0be2d66b73c00e10897fedfb229c528f94566737cc36cf3069f34555dec` |
+| `390x844-selection.png` | `f820f2a2a22b20fe20756d282c4adc6e65bb826871f972747d4b0f3e249eb55e` |
+| `1440x900-catalog.png` | `b0143c982d439fdb6aec923bc52761de7b8ba5d87910bb4b29e9fc827eefe1d0` |
+| `1440x900-detail.png` | `8f876daaa966000e0fb083f5c0446893184f95beb182c03409a7209cd953f2d4` |
+| `1440x900-history.png` | `53cab0b995f030355add993cdfd01a08f6c77a85edc8b7708fa4fccdc8c740ea` |
+| `1440x900-selection.png` | `f572b203dd0b18ac9373023deba61bb0359753243c01566292a7309cfc1c12bf` |
+| `browser-results.json` | `e9de4eba8ef4690731536ce284adabc392598c9533ac5ebb51d5402ef878c0f1` |
+| `axe-results.json` | `c8be5cc052b0a3c6ce544c279bf2ca9882118df1b6aab604396e31c77763a7e7` |
+| `live-source-results.json` | `8ec68cea621e6a77cf6dd588093f67320f0bc604b4d3b7029e2140831ff4aa21` |
+
+이 evidence는 production platform, production database, human publication approval, full-catalog
+historical coverage, deployment·traffic switch, domain·DNS, trademark clearance를 증명하지 않는다.
diff --git a/output/playwright/vnext-redesign-v2/axe-results.json b/output/playwright/vnext-redesign-v2/axe-results.json
new file mode 100644
index 0000000..e2403df
--- /dev/null
+++ b/output/playwright/vnext-redesign-v2/axe-results.json
@@ -0,0 +1,19 @@
+{
+  "axeSource": ".cache/axe/axe.min.js",
+  "axeVersion": "4.13.0",
+  "fixtureKind": "live-public-api-normalized-test-publication",
+  "outputDirectory": "output/playwright/vnext-redesign-v2",
+  "scanCount": 9,
+  "scans": [
+    {"name": "catalog-ready", "passes": 23, "reviewedContrastNodes": 12, "viewport": "390x844"},
+    {"name": "detail-ready", "passes": 21, "reviewedContrastNodes": 4, "viewport": "390x844"},
+    {"name": "history-ready", "passes": 26, "reviewedContrastNodes": 7, "viewport": "390x844"},
+    {"name": "regions-ready", "passes": 25, "reviewedContrastNodes": 2, "viewport": "390x844"},
+    {"name": "markets-ready", "passes": 25, "reviewedContrastNodes": 1, "viewport": "390x844"},
+    {"name": "selection-ready", "passes": 25, "reviewedContrastNodes": 3, "viewport": "390x844"},
+    {"name": "catalog-validation", "passes": 27, "reviewedContrastNodes": 1, "viewport": "390x844"},
+    {"name": "catalog-server-error", "passes": 19, "reviewedContrastNodes": 0, "viewport": "390x844"},
+    {"name": "generic-not-found", "passes": 16, "reviewedContrastNodes": 0, "viewport": "390x844"}
+  ],
+  "viewport": "390x844"
+}
diff --git a/output/playwright/vnext-redesign-v2/browser-results.json b/output/playwright/vnext-redesign-v2/browser-results.json
new file mode 100644
index 0000000..de73d64
--- /dev/null
+++ b/output/playwright/vnext-redesign-v2/browser-results.json
@@ -0,0 +1,30 @@
+{
+  "candidateCommit": "d97888885e8a2e5b8db88005ddf0bf3a336dcdc6",
+  "externalRequests": 0,
+  "fixtureKind": "live-public-api-normalized-test-publication",
+  "generatedAt": "2026-08-31T07:50:19Z",
+  "outputDirectory": "output/playwright/vnext-redesign-v2",
+  "overflowOnlyViewports": [
+    "360x800",
+    "768x1024"
+  ],
+  "readyFlowViewports": [
+    "390x844",
+    "1440x900"
+  ],
+  "screenshotCount": 8,
+  "screenshotSurfaces": [
+    "catalog",
+    "detail",
+    "history",
+    "selection"
+  ],
+  "surfaces": [
+    "catalog",
+    "detail",
+    "history",
+    "regions",
+    "markets",
+    "selection"
+  ]
+}
diff --git a/output/playwright/vnext-redesign-v2/live-source-results.json b/output/playwright/vnext-redesign-v2/live-source-results.json
new file mode 100644
index 0000000..e5da217
--- /dev/null
+++ b/output/playwright/vnext-redesign-v2/live-source-results.json
@@ -0,0 +1,11 @@
+{
+  "fixtureKind": "live-public-api-normalized-test-publication",
+  "marketRows": 9,
+  "monthlyRows": 36,
+  "rawResponseRetained": false,
+  "recentRows": 10,
+  "regionalRows": 1,
+  "sourceCallsDuringSsr": 0,
+  "ssrRoutes": 5,
+  "status": "PASS"
+}
diff --git a/scripts/vnext_redesign_v2_axe_acceptance.js b/scripts/vnext_redesign_v2_axe_acceptance.js
new file mode 100644
index 0000000..841d6cf
--- /dev/null
+++ b/scripts/vnext_redesign_v2_axe_acceptance.js
@@ -0,0 +1,249 @@
+async (page) => {
+  const baseUrl = "http://127.0.0.1:8000";
+  const outputDirectory = "output/playwright/vnext-redesign-v2";
+  const requiredAxeVersion = "4.13.0";
+  const expectedScanCount = 9;
+  const configuredAxePath =
+    typeof process !== "undefined" && process.env && process.env.AXE_CORE_PATH
+      ? process.env.AXE_CORE_PATH
+      : ".cache/axe/axe.min.js";
+  const externalRequests = [];
+
+  function assert(condition, message) {
+    if (!condition) throw new Error(message);
+  }
+
+  assert(configuredAxePath.length > 0, "AXE_CORE_PATH must not be empty");
+  assert(
+    !/^(?:https?:)?\/\//i.test(configuredAxePath),
+    "AXE_CORE_PATH must be a local filesystem path; CDN injection is prohibited",
+  );
+  await page.addInitScript({ path: configuredAxePath });
+  page.on("request", (request) => {
+    const requestUrl = request.url();
+    if (!requestUrl.startsWith("http://") && !requestUrl.startsWith("https://")) return;
+    if (requestUrl !== baseUrl && !requestUrl.startsWith(`${baseUrl}/`)) {
+      externalRequests.push("external-origin");
+    }
+  });
+
+  function relativeUrl(value) {
+    assert(typeof value === "string", "axe navigation target must be a string");
+    if (value === baseUrl) return "/";
+    if (value.startsWith(`${baseUrl}/`)) return value.slice(baseUrl.length);
+    assert(
+      value.startsWith("/") && !value.startsWith("//") && !/[\r\n]/.test(value),
+      "axe navigation escaped the loopback origin",
+    );
+    return value;
+  }
+
+  async function goto(path, expectedStatus, label) {
+    let response;
+    try {
+      response = await page.goto(`${baseUrl}${path}`, { waitUntil: "networkidle" });
+    } catch (_error) {
+      throw new Error(`${label} navigation failed`);
+    }
+    assert(response !== null, `${label} navigation response missing`);
+    assert(
+      response.status() === expectedStatus,
+      `${label} returned ${response.status()}, expected ${expectedStatus}`,
+    );
+    const headers = await response.allHeaders();
+    assert(
+      (headers["cache-control"] || "")
+        .split(",")
+        .some((directive) => directive.trim().toLowerCase() === "no-store"),
+      `${label} must be no-store`,
+    );
+    await page.evaluate(async () => {
+      if (document.fonts) await document.fonts.ready;
+    });
+    return response;
+  }
+
+  async function resolveHistoricalDetailPath() {
+    const candidates = await page.locator(".ledger-entry__link").evaluateAll((links) =>
+      links
+        .slice(0, 30)
+        .map((link) => link.getAttribute("href"))
+        .filter((value) => typeof value === "string" && value.length > 0),
+    );
+    assert(candidates.length >= 1, "axe catalog has no detail candidates");
+    for (let index = 0; index < candidates.length; index += 1) {
+      const response = await goto(
+        relativeUrl(candidates[index]),
+        200,
+        `axe historical candidate ${index + 1}`,
+      );
+      const headers = await response.allHeaders();
+      const historical = headers["x-historical-publication-fact-set"] || "";
+      const hasHistoricalHeader = /^[0-9a-f]{64}$/.test(historical);
+      const hasHistoryNavigation =
+        (await page.getByRole("link", { name: "월별 기록", exact: true }).count()) === 1;
+      assert(
+        hasHistoricalHeader === hasHistoryNavigation,
+        `axe historical candidate ${index + 1} navigation/header mismatch`,
+      );
+      if (hasHistoricalHeader) return relativeUrl(page.url());
+    }
+    throw new Error("axe bounded candidates contain no historical publication");
+  }
+
+  async function scan(name, path, expectedStatus = 200) {
+    await goto(path, expectedStatus, name);
+    const report = await page.evaluate(async (expectedVersion) => {
+      if (!window.axe) throw new Error("local axe-core did not load");
+      if (window.axe.version !== expectedVersion) {
+        throw new Error(`axe-core ${expectedVersion} required, received ${window.axe.version}`);
+      }
+      window.axe.configure({ rules: [{ id: "target-size", enabled: true }] });
+      const report = await window.axe.run(document, {
+        resultTypes: ["violations", "incomplete", "passes"],
+        runOnly: {
+          type: "tag",
+          values: ["wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "wcag22aa"],
+        },
+      });
+      const reviewedIncomplete = [];
+      const unexpectedIncomplete = [];
+      for (const result of report.incomplete) {
+        const isReviewedContrast =
+          result.id === "color-contrast" &&
+          result.nodes.every((node) =>
+            node.target
+              .flat(Number.POSITIVE_INFINITY)
+              .filter((selector) => typeof selector === "string")
+              .every((selector) => {
+                const element = document.querySelector(selector);
+                return (
+                  element?.getAttribute("aria-hidden") === "true" ||
+                  element?.matches("text.history-chart__label") === true
+                );
+              }),
+          );
+        (isReviewedContrast ? reviewedIncomplete : unexpectedIncomplete).push(result);
+      }
+      return {
+        incomplete: unexpectedIncomplete,
+        passes: report.passes,
+        reviewedIncomplete,
+        violations: report.violations,
+      };
+    }, requiredAxeVersion);
+    const violations = report.violations.map((violation) => ({
+      help: violation.help,
+      id: violation.id,
+      impact: violation.impact,
+      nodeCount: violation.nodes.length,
+    }));
+    const incomplete = report.incomplete.map((result) => ({
+      id: result.id,
+      impact: result.impact,
+      nodeCount: result.nodes.length,
+    }));
+    assert(violations.length === 0, `${name} axe violations: ${JSON.stringify(violations)}`);
+    assert(
+      incomplete.length === 0,
+      `${name} axe incomplete results require review: ${JSON.stringify(incomplete)}`,
+    );
+    return {
+      name,
+      passes: report.passes.length,
+      reviewedContrastNodes: report.reviewedIncomplete.reduce(
+        (total, result) => total + result.nodes.length,
+        0,
+      ),
+      viewport: "390x844",
+    };
+  }
+
+  await page.setViewportSize({ width: 390, height: 844 });
+  const scans = [];
+  scans.push(await scan("catalog-ready", "/"));
+  const detailPath = await resolveHistoricalDetailPath();
+
+  scans.push(await scan("detail-ready", detailPath));
+  const historyPath = await page
+    .getByRole("link", { name: "월별 기록", exact: true })
+    .getAttribute("href");
+  assert(historyPath !== null, "axe history navigation missing");
+  await goto(relativeUrl(historyPath), 200, "history-region-selection");
+  const regionSelect = page.locator("#history-region");
+  const regionValue = await regionSelect
+    .locator('option[value]:not([value=""])')
+    .first()
+    .getAttribute("value");
+  assert(regionValue !== null, "axe history region option missing");
+  await regionSelect.selectOption(regionValue);
+  const [historyResponse] = await Promise.all([
+    page.waitForNavigation({ waitUntil: "networkidle" }),
+    page.getByRole("button", { name: "월별 기록 보기", exact: true }).click(),
+  ]);
+  assert(historyResponse !== null && historyResponse.status() === 200, "axe history submit failed");
+  const readyHistoryPath = relativeUrl(page.url());
+  scans.push(await scan("history-ready", readyHistoryPath));
+
+  const regionsPath = await page
+    .getByRole("link", { name: "지역별 조사값", exact: true })
+    .getAttribute("href");
+  assert(regionsPath !== null, "axe regions navigation missing");
+  scans.push(await scan("regions-ready", relativeUrl(regionsPath)));
+  const marketsPath = await page
+    .getByRole("link", { name: /시장별 값 보기/ })
+    .first()
+    .getAttribute("href");
+  assert(marketsPath !== null, "axe markets navigation missing");
+  scans.push(await scan("markets-ready", relativeUrl(marketsPath)));
+
+  await goto(detailPath, 200, "detail-selection-link");
+  const selectionPath = await page
+    .getByRole("link", { name: "선택 목록에 담기", exact: true })
+    .getAttribute("href");
+  assert(selectionPath !== null, "axe selection navigation missing");
+  await goto(relativeUrl(selectionPath), 200, "selection-add-form");
+  const candidateSelect = page.locator("#selection-add-item");
+  const candidateValue = await candidateSelect
+    .locator('option[value]:not([value=""])')
+    .first()
+    .getAttribute("value");
+  assert(candidateValue !== null, "axe selection candidate missing");
+  await candidateSelect.selectOption(candidateValue);
+  const [selectionResponse] = await Promise.all([
+    page.waitForNavigation({ waitUntil: "networkidle" }),
+    page.getByRole("button", { name: "선택 목록에 추가", exact: true }).click(),
+  ]);
+  assert(
+    selectionResponse !== null && selectionResponse.status() === 200,
+    "axe selection add failed",
+  );
+  const readySelectionPath = relativeUrl(page.url());
+  scans.push(await scan("selection-ready", readySelectionPath));
+  assert((await page.locator(".selection-row").count()) === 2, "axe selection row count invalid");
+  const selectedItemCount = await page.evaluate(
+    () => new URLSearchParams(window.location.search).getAll("series").length,
+  );
+  assert(selectedItemCount === 2, "axe selection URL item count invalid");
+
+  scans.push(await scan("catalog-validation", "/?page=01", 400));
+  scans.push(
+    await scan("catalog-server-error", "/__qa__/catalog/server_error/", 503),
+  );
+  scans.push(await scan("generic-not-found", "/__qa__/catalog/error_404/", 404));
+
+  assert(
+    scans.length === expectedScanCount,
+    `redesign v2 axe scan matrix incomplete: ${scans.length}/${expectedScanCount}`,
+  );
+  assert(externalRequests.length === 0, `external request count: ${externalRequests.length}`);
+  return {
+    axeSource: configuredAxePath,
+    axeVersion: requiredAxeVersion,
+    fixtureKind: "live-public-api-normalized-test-publication",
+    outputDirectory,
+    scanCount: scans.length,
+    scans,
+    viewport: "390x844",
+  };
+}
diff --git a/scripts/vnext_redesign_v2_browser_acceptance.js b/scripts/vnext_redesign_v2_browser_acceptance.js
new file mode 100644
index 0000000..6ede1c3
--- /dev/null
+++ b/scripts/vnext_redesign_v2_browser_acceptance.js
@@ -0,0 +1,492 @@
+async (page) => {
+  const baseUrl = "http://127.0.0.1:8000";
+  const outputDirectory = "output/playwright/vnext-redesign-v2";
+  const screenshotPaths = [];
+  const consoleErrors = [];
+  const externalRequests = [];
+  const failedRequests = [];
+  const failedSubresources = [];
+  let recentFactSet = null;
+  let historicalFactSet = null;
+
+  page.on("console", (message) => {
+    if (message.type() === "error") consoleErrors.push("console-error");
+  });
+  page.on("request", (request) => {
+    const requestUrl = request.url();
+    if (!requestUrl.startsWith("http://") && !requestUrl.startsWith("https://")) return;
+    if (requestUrl !== baseUrl && !requestUrl.startsWith(`${baseUrl}/`)) {
+      externalRequests.push("external-origin");
+    }
+  });
+  page.on("requestfailed", (request) => {
+    failedRequests.push(request.resourceType());
+  });
+  page.on("response", (response) => {
+    const request = response.request();
+    if (response.status() < 400 || request.resourceType() === "document") return;
+    failedSubresources.push({
+      resourceType: request.resourceType(),
+      status: response.status(),
+    });
+  });
+
+  function assert(condition, message) {
+    if (!condition) throw new Error(message);
+  }
+
+  function relativeUrl(value) {
+    assert(typeof value === "string", "navigation target must be a string");
+    if (value === baseUrl) return "/";
+    if (value.startsWith(`${baseUrl}/`)) return value.slice(baseUrl.length);
+    assert(
+      value.startsWith("/") && !value.startsWith("//") && !/[\r\n]/.test(value),
+      "navigation escaped the loopback origin",
+    );
+    return value;
+  }
+
+  async function waitForFonts() {
+    await page.evaluate(async () => {
+      if (document.fonts) await document.fonts.ready;
+    });
+  }
+
+  async function assertResponse(response, label, scope = "recent", expectedStatus = 200) {
+    assert(response !== null, `${label} navigation response missing`);
+    assert(
+      response.status() === expectedStatus,
+      `${label} returned ${response.status()}, expected ${expectedStatus}`,
+    );
+    const headers = await response.allHeaders();
+    const cacheDirectives = (headers["cache-control"] || "")
+      .split(",")
+      .map((directive) => directive.trim().split("=", 1)[0].toLowerCase());
+    assert(cacheDirectives.includes("no-store"), `${label} must be no-store`);
+    assert(headers["referrer-policy"] === "no-referrer", `${label} referrer policy changed`);
+    assert(
+      (headers["content-security-policy"] || "").includes("script-src 'none'"),
+      `${label} script CSP changed`,
+    );
+    assert(headers["x-content-type-options"] === "nosniff", `${label} nosniff missing`);
+    assert(headers["x-frame-options"] === "DENY", `${label} frame boundary changed`);
+    assert(!("set-cookie" in headers), `${label} must not create a session cookie`);
+
+    const recent = headers["x-publication-fact-set"] || "";
+    assert(/^[0-9a-f]{64}$/.test(recent), `${label} recent fact-set header missing`);
+    if (recentFactSet === null) recentFactSet = recent;
+    assert(recent === recentFactSet, `${label} recent publication changed during the flow`);
+
+    if (scope === "both") {
+      const historical = headers["x-historical-publication-fact-set"] || "";
+      assert(/^[0-9a-f]{64}$/.test(historical), `${label} historical fact-set header missing`);
+      if (historicalFactSet === null) historicalFactSet = historical;
+      assert(
+        historical === historicalFactSet,
+        `${label} historical publication changed during the flow`,
+      );
+    } else {
+      assert(
+        !("x-historical-publication-fact-set" in headers),
+        `${label} unexpectedly mixed historical publication state`,
+      );
+    }
+    await waitForFonts();
+    return headers;
+  }
+
+  async function goto(path, label, scope = "recent", expectedStatus = 200) {
+    let response;
+    try {
+      response = await page.goto(`${baseUrl}${path}`, { waitUntil: "networkidle" });
+    } catch (_error) {
+      throw new Error(`${label} navigation failed`);
+    }
+    await assertResponse(response, label, scope, expectedStatus);
+    return response;
+  }
+
+  async function follow(locator, label, scope = "recent") {
+    assert((await locator.count()) === 1, `${label} must have one navigation target`);
+    const [response] = await Promise.all([
+      page.waitForNavigation({ waitUntil: "networkidle" }),
+      locator.click(),
+    ]);
+    await assertResponse(response, label, scope);
+    return relativeUrl(page.url());
+  }
+
+  async function submit(button, label, scope = "recent") {
+    const [response] = await Promise.all([
+      page.waitForNavigation({ waitUntil: "networkidle" }),
+      button.click(),
+    ]);
+    await assertResponse(response, label, scope);
+    return relativeUrl(page.url());
+  }
+
+  async function resolveHistoricalDetailPath(label) {
+    const candidates = await page.locator(".ledger-entry__link").evaluateAll((links) =>
+      links
+        .slice(0, 30)
+        .map((link) => link.getAttribute("href"))
+        .filter((value) => typeof value === "string" && value.length > 0),
+    );
+    assert(candidates.length >= 1, `${label} has no catalog detail candidates`);
+    for (let index = 0; index < candidates.length; index += 1) {
+      let response;
+      try {
+        response = await page.goto(`${baseUrl}${relativeUrl(candidates[index])}`, {
+          waitUntil: "networkidle",
+        });
+      } catch (_error) {
+        throw new Error(`${label} candidate ${index + 1} navigation failed`);
+      }
+      assert(response !== null, `${label} candidate ${index + 1} response missing`);
+      const candidateHeaders = await response.allHeaders();
+      const hasHistoricalHeader = "x-historical-publication-fact-set" in candidateHeaders;
+      await assertResponse(
+        response,
+        `${label} candidate ${index + 1}`,
+        hasHistoricalHeader ? "both" : "recent",
+      );
+      const hasHistoryNavigation =
+        (await page.getByRole("link", { name: "월별 기록", exact: true }).count()) === 1;
+      assert(
+        hasHistoricalHeader === hasHistoryNavigation,
+        `${label} candidate ${index + 1} historical navigation/header mismatch`,
+      );
+      if (hasHistoricalHeader) return relativeUrl(page.url());
+    }
+    throw new Error(`${label} bounded candidates contain no historical publication`);
+  }
+
+  async function assertNoOverflow(label) {
+    const dimensions = await page.evaluate(() => ({
+      clientWidth: document.documentElement.clientWidth,
+      scrollWidth: document.documentElement.scrollWidth,
+    }));
+    assert(
+      dimensions.scrollWidth <= dimensions.clientWidth,
+      `${label} horizontal overflow ${dimensions.scrollWidth}/${dimensions.clientWidth}`,
+    );
+  }
+
+  async function assertDocumentContract(label) {
+    const result = await page.evaluate(() => {
+      const targets = [
+        ...document.querySelectorAll(
+          'a, button, input:not([type="hidden"]), select, summary',
+        ),
+      ]
+        .map((element) => ({ element, rectangle: element.getBoundingClientRect() }))
+        .filter(({ rectangle }) => rectangle.width > 0 && rectangle.height > 0);
+      const undersized = targets
+        .filter(({ rectangle }) => rectangle.width < 44 || rectangle.height < 44)
+        .map(({ element, rectangle }) => ({
+          height: Math.round(rectangle.height),
+          tag: element.tagName,
+          width: Math.round(rectangle.width),
+        }));
+      const externalResources = [
+        ...document.querySelectorAll("img[src], link[href], source[src], source[srcset]"),
+      ]
+        .map(
+          (element) =>
+            element.getAttribute("src") ||
+            element.getAttribute("srcset") ||
+            element.getAttribute("href"),
+        )
+        .filter(Boolean)
+        .filter((value) => {
+          const resolved = new URL(value, document.baseURI);
+          return (
+            ["http:", "https:"].includes(resolved.protocol) &&
+            resolved.origin !== location.origin
+          );
+        });
+      const eventHandlers = [...document.querySelectorAll("*")].flatMap((element) =>
+        [...element.attributes]
+          .filter((attribute) => attribute.name.toLowerCase().startsWith("on"))
+          .map((attribute) => `${element.tagName.toLowerCase()}[${attribute.name}]`),
+      );
+      return {
+        eventHandlers,
+        externalResources,
+        h1Count: document.querySelectorAll("h1").length,
+        lang: document.documentElement.lang,
+        mainCount: document.querySelectorAll("main").length,
+        positiveTabIndexes: [...document.querySelectorAll("[tabindex]")]
+          .map((element) => Number(element.getAttribute("tabindex")))
+          .filter((value) => value > 0),
+        scripts: document.scripts.length,
+        undersized,
+      };
+    });
+    await assertNoOverflow(label);
+    assert(result.mainCount === 1, `${label} must have one main landmark`);
+    assert(result.h1Count === 1, `${label} must have one h1`);
+    assert(result.lang === "ko", `${label} must declare Korean language`);
+    assert(result.scripts === 0, `${label} must remain server-rendered without scripts`);
+    assert(
+      result.eventHandlers.length === 0,
+      `${label} inline event handlers ${JSON.stringify(result.eventHandlers)}`,
+    );
+    assert(result.positiveTabIndexes.length === 0, `${label} must keep natural keyboard order`);
+    assert(
+      result.undersized.length === 0,
+      `${label} undersized targets ${JSON.stringify(result.undersized)}`,
+    );
+    assert(
+      result.externalResources.length === 0,
+      `${label} external resources ${JSON.stringify(result.externalResources)}`,
+    );
+  }
+
+  async function assertSkipLink(label) {
+    await page.evaluate(() => {
+      if (document.activeElement instanceof HTMLElement) document.activeElement.blur();
+      window.scrollTo(0, 0);
+    });
+    await page.keyboard.press("Tab");
+    const focus = await page.evaluate(() => {
+      const element = document.activeElement;
+      if (!(element instanceof HTMLElement)) return null;
+      const style = getComputedStyle(element);
+      const rectangle = element.getBoundingClientRect();
+      return {
+        className: element.className,
+        focusVisible: element.matches(":focus-visible"),
+        height: rectangle.height,
+        outlineStyle: style.outlineStyle,
+        outlineWidth: style.outlineWidth,
+        top: rectangle.top,
+        width: rectangle.width,
+      };
+    });
+    assert(focus !== null && focus.className.includes("skip-link"), `${label} skip link not first`);
+    assert(focus.focusVisible, `${label} skip link is not visibly keyboard-focused`);
+    assert(
+      focus.outlineStyle !== "none" && focus.outlineWidth !== "0px",
+      `${label} skip link focus ring missing`,
+    );
+    assert(
+      focus.top >= 0 && focus.width >= 44 && focus.height >= 44,
+      `${label} focused skip link is not visible and touch-sized`,
+    );
+    await page.keyboard.press("Enter");
+    assert(
+      (await page.locator("#main-content:focus").count()) === 1,
+      `${label} skip target missing`,
+    );
+  }
+
+  async function tabUntil(selector, label, maximumTabs = 64) {
+    await page.evaluate(() => {
+      if (document.activeElement instanceof HTMLElement) document.activeElement.blur();
+    });
+    for (let index = 0; index < maximumTabs; index += 1) {
+      await page.keyboard.press("Tab");
+      const matched = await page.evaluate(
+        (target) =>
+          document.activeElement instanceof Element && document.activeElement.matches(target),
+        selector,
+      );
+      if (matched) return;
+    }
+    throw new Error(`${label} keyboard target was not reached`);
+  }
+
+  async function assertFocusedLink(label) {
+    const focus = await page.evaluate(() => {
+      const element = document.activeElement;
+      if (!(element instanceof HTMLAnchorElement)) return null;
+      const rectangle = element.getBoundingClientRect();
+      const style = getComputedStyle(element);
+      return {
+        focusVisible: element.matches(":focus-visible"),
+        height: rectangle.height,
+        outlineStyle: style.outlineStyle,
+        outlineWidth: style.outlineWidth,
+        width: rectangle.width,
+      };
+    });
+    assert(focus !== null, `${label} catalog item link did not receive focus`);
+    assert(focus.focusVisible, `${label} catalog item link is not focus-visible`);
+    assert(
+      focus.outlineStyle !== "none" && focus.outlineWidth !== "0px",
+      `${label} catalog item link focus outline missing`,
+    );
+    assert(
+      focus.width >= 44 && focus.height >= 44,
+      `${label} focused catalog item link is not touch-sized`,
+    );
+  }
+
+  async function capture(name) {
+    assert(screenshotPaths.length < 8, "redesign v2 screenshot budget exceeded");
+    const path = `${outputDirectory}/${name}.png`;
+    await page.screenshot({ path, fullPage: true });
+    screenshotPaths.push(path);
+  }
+
+  async function runReadyFlow(viewport, knownDetailPath = null) {
+    const mobileEvidence = viewport.width === 390;
+    await page.setViewportSize({ width: viewport.width, height: viewport.height });
+
+    await goto("/", `${viewport.label} catalog`);
+    assert((await page.locator(".ledger-entry").count()) >= 2, `${viewport.label} catalog incomplete`);
+    await assertDocumentContract(`${viewport.label} catalog`);
+    if (mobileEvidence) {
+      const firstRecord = await page.locator(".ledger-row").first().boundingBox();
+      assert(firstRecord !== null, `${viewport.label} first catalog record missing`);
+      assert(firstRecord.y >= 0, `${viewport.label} first catalog record begins above the fold`);
+      assert(
+        firstRecord.y + firstRecord.height <= viewport.height,
+        `${viewport.label} first catalog record is not fully visible before scrolling`,
+      );
+    }
+    await capture(`${viewport.label}-catalog`);
+    await assertSkipLink(`${viewport.label} catalog`);
+
+    const detailPath =
+      knownDetailPath || (await resolveHistoricalDetailPath(`${viewport.label} historical detail`));
+    await goto("/", `${viewport.label} catalog keyboard restart`);
+    const detailSelector = `.ledger-entry__link[href="${detailPath}"]`;
+    assert((await page.locator(detailSelector).count()) === 1, `${viewport.label} detail link missing`);
+    await tabUntil(detailSelector, `${viewport.label} historical catalog item`);
+    await assertFocusedLink(`${viewport.label} historical catalog item`);
+    const [detailResponse] = await Promise.all([
+      page.waitForNavigation({ waitUntil: "networkidle" }),
+      page.keyboard.press("Enter"),
+    ]);
+    await assertResponse(detailResponse, `${viewport.label} detail keyboard navigation`, "both");
+    assert(relativeUrl(page.url()) === detailPath, `${viewport.label} detail target changed`);
+    assert(
+      (await page.locator(".comparison-ledger").count()) === 1,
+      `${viewport.label} detail missing`,
+    );
+    await assertDocumentContract(`${viewport.label} detail`);
+    await capture(`${viewport.label}-detail`);
+
+    const historyPath = await follow(
+      page.getByRole("link", { name: "월별 기록", exact: true }),
+      `${viewport.label} history selection`,
+      "both",
+    );
+    assert(historyPath.startsWith(`${detailPath}history/`), `${viewport.label} history URL changed`);
+    const regionSelect = page.locator("#history-region");
+    const regionValue = await regionSelect
+      .locator('option[value]:not([value=""])')
+      .first()
+      .getAttribute("value");
+    assert(regionValue !== null, `${viewport.label} history region option missing`);
+    await regionSelect.selectOption(regionValue);
+    const readyHistoryPath = await submit(
+      page.getByRole("button", { name: "월별 기록 보기", exact: true }),
+      `${viewport.label} history ready`,
+      "both",
+    );
+    assert((await page.locator(".history-chart").count()) === 1, `${viewport.label} chart missing`);
+    assert((await page.locator(".month-row").count()) >= 1, `${viewport.label} month rows missing`);
+    await assertDocumentContract(`${viewport.label} history`);
+    await capture(`${viewport.label}-history`);
+
+    const regionsPath = await follow(
+      page.getByRole("link", { name: "지역별 조사값", exact: true }),
+      `${viewport.label} regions`,
+      "both",
+    );
+    assert((await page.locator(".region-row").count()) >= 1, `${viewport.label} regions empty`);
+    await assertDocumentContract(`${viewport.label} regions`);
+
+    const marketsPath = await follow(
+      page.getByRole("link", { name: /시장별 값 보기/ }).first(),
+      `${viewport.label} markets`,
+      "both",
+    );
+    assert((await page.locator(".market-row").count()) >= 1, `${viewport.label} markets empty`);
+    await assertDocumentContract(`${viewport.label} markets`);
+
+    await follow(
+      page.getByRole("link", { name: "최근 조사값", exact: true }),
+      `${viewport.label} detail return`,
+      "both",
+    );
+    await follow(
+      page.getByRole("link", { name: "선택 목록에 담기", exact: true }),
+      `${viewport.label} selection first item`,
+    );
+    assert(
+      (await page.locator(".selection-row").count()) === 1,
+      `${viewport.label} first selection missing`,
+    );
+    const candidateValue = await page
+      .locator("#selection-add-item")
+      .locator('option[value]:not([value=""])')
+      .first()
+      .getAttribute("value");
+    assert(candidateValue !== null, `${viewport.label} selection candidate missing`);
+    await page.locator("#selection-add-item").selectOption(candidateValue);
+    const selectionPath = await submit(
+      page.getByRole("button", { name: "선택 목록에 추가", exact: true }),
+      `${viewport.label} selection add`,
+    );
+    assert((await page.locator(".selection-row").count()) === 2, `${viewport.label} add failed`);
+    const selectedItemCount = await page.evaluate(
+      () => new URLSearchParams(window.location.search).getAll("series").length,
+    );
+    assert(selectedItemCount === 2, `${viewport.label} selection URL must carry two items`);
+    await assertDocumentContract(`${viewport.label} selection`);
+    await capture(`${viewport.label}-selection`);
+
+    return [
+      { name: "catalog", path: "/", scope: "recent" },
+      { name: "detail", path: detailPath, scope: "both" },
+      { name: "history", path: readyHistoryPath, scope: "both" },
+      { name: "regions", path: regionsPath, scope: "both" },
+      { name: "markets", path: marketsPath, scope: "both" },
+      { name: "selection", path: selectionPath, scope: "recent" },
+    ];
+  }
+
+  const mobilePaths = await runReadyFlow({ width: 390, height: 844, label: "390x844" });
+  for (const viewport of [
+    { width: 360, height: 800, label: "360x800" },
+    { width: 768, height: 1024, label: "768x1024" },
+  ]) {
+    await page.setViewportSize({ width: viewport.width, height: viewport.height });
+    for (const surface of mobilePaths) {
+      await goto(surface.path, `${viewport.label} ${surface.name} overflow`, surface.scope);
+      await assertNoOverflow(`${viewport.label} ${surface.name}`);
+    }
+  }
+  const historicalDetailSurface = mobilePaths.find((surface) => surface.name === "detail");
+  assert(historicalDetailSurface !== undefined, "historical detail surface missing");
+  await runReadyFlow(
+    { width: 1440, height: 900, label: "1440x900" },
+    historicalDetailSurface.path,
+  );
+
+  assert(
+    screenshotPaths.length === 8,
+    `redesign v2 screenshot matrix incomplete: ${screenshotPaths.length}/8`,
+  );
+  assert(consoleErrors.length === 0, `browser console error count: ${consoleErrors.length}`);
+  assert(externalRequests.length === 0, `external request count: ${externalRequests.length}`);
+  assert(failedRequests.length === 0, `failed request count: ${failedRequests.length}`);
+  assert(
+    failedSubresources.length === 0,
+    `failed subresource count: ${failedSubresources.length}`,
+  );
+  return {
+    externalRequests: 0,
+    fixtureKind: "live-public-api-normalized-test-publication",
+    outputDirectory,
+    overflowOnlyViewports: ["360x800", "768x1024"],
+    readyFlowViewports: ["390x844", "1440x900"],
+    screenshotCount: screenshotPaths.length,
+    screenshotSurfaces: ["catalog", "detail", "history", "selection"],
+    surfaces: ["catalog", "detail", "history", "regions", "markets", "selection"],
+  };
+}


